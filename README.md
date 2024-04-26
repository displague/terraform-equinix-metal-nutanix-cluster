# Nutanix Cluster on Equinix Metal

This Terraform module will deploy a proof-of-concept demonstrative Nutanix Cluster in Layer 2 isolation on Equinix Metal. DNS, DHCP, and Cluster internet access is managed by an Ubuntu 22.04 bastion/gateway node.

## Acronyms and Terms

- AOS: Acropolis Operating System
- NOS: Nutanix Operating System (Used interchangeably with AOS)
- AHV: AOS Hypervisor
- Phoenix: The AOS/NOS Installer
- CVM: Cluster Virtual Machine
- Prism: AOS Cluster Web UI

## Nutanix Installation in a Nutshell

For those who are unfamiliar with Nutanix. Nutanix is HCI (Hyperconverged Infrastructure) software. See [https://www.nutanix.com/products/nutanix-cloud-infrastructure](https://www.nutanix.com/products/nutanix-cloud-infrastructure) for more details from Nutanix.

Nutanix AOS is typically deployed in a private network without public IPs assigned directly to the host.
This experience differs from what many cloud users would expect in an OS deployment.

This POC Terraform module is inspired by the [Deploying a multi-node Nutanix cluster on Metal](https://deploy.equinix.com/developers/guides/deploying-a-multi-node-nutanix-cluster-on-equinix-metal/) guide which goes into detail about how to deploy Nutanix and the required networking configuration on Equinix Metal. Follow that guide for step-by-step instructions that you can customize along the way.

By deploying this POC Terraform module, you will get an automated and opinionated minimal Nutanix Cluster that will help provide a quick introduction to the platform's capabilities.

This POC is not intended to be used as a demonstration of best practices or Day-2 operations, including security, scale, monitoring, and disaster recovery.

To accommodate deployment requirements, this module will create:

- 1x [c3.small.x86](https://deploy.equinix.com/product/servers/c3-small/) node running [Ubuntu 22.04](https://deploy.equinix.com/developers/docs/metal/operating-systems/supported/#ubuntu) in a [hybrid-bonded networking mode](https://deploy.equinix.com/developers/docs/metal/layer2-networking/hybrid-bonded-mode/)

  This "bastion" node will act as a router and jump box. DHCP, DNS, and NAT (internet access) functionality will be provided by [`dnsmasq`](https://dnsmasq.org/doc.html).

- 3x [m3.large.x86](https://deploy.equinix.com/product/servers/m3-large/) nodes running [Nutanix LTS 6.5](https://deploy.equinix.com/developers/docs/metal/operating-systems/licensed/#nutanix-cloud-platform-on-equinix-metal) in [layer2-bonded networking mode](https://deploy.equinix.com/developers/docs/metal/layer2-networking/layer2-bonded-mode/)

  [Workload Optimized](https://deploy.equinix.com/developers/docs/metal/hardware/workload-optimized-plans/) [hardware reservations](https://deploy.equinix.com/developers/docs/metal/deploy/reserved/) are preferred and required for [capacity](https://deploy.equinix.com/developers/docs/metal/locations/capacity/) and to ensure hardware compatibility with Nutanix. On-demand instances will be deployed by default, see ["On-Demand Instances"](#on-demand-instances) notes below for more details.

- 1x [VLAN](https://deploy.equinix.com/developers/docs/metal/layer2-networking/vlans/) and [Metal Gateway](https://deploy.equinix.com/developers/docs/metal/layer2-networking/metal-gateway/) with [VRF](https://deploy.equinix.com/developers/docs/metal/layer2-networking/vrf/)

  The VRF will route a `/22` IP range within the VLAN, providing ample IP space for POC purposes.

  The bastion node will attach to this VLAN, and the Nutanix nodes will passively access this as their [Native VLAN](https://deploy.equinix.com/developers/docs/metal/layer2-networking/native-vlan/) with DHCP addresses from the VRF space assigned by the bastion node.

- 1x [SSH Key](https://deploy.equinix.com/developers/docs/metal/identity-access-management/ssh-keys/) configured to access the bastion node

  Terraform will create an SSH key scoped to this deployment. The key will be stored in the Terraform workspace.

- 1x [Metal Project](https://deploy.equinix.com/developers/docs/metal/projects/creating-a-project/)

  Optionally deploy a new project to test the POC in isolation or deploy it within an existing project.

## Terraform installation

You'll need [Terraform installed](https://developer.hashicorp.com/terraform/install) and an [Equinix Metal account](https://deploy.equinix.com/developers/docs/metal/identity-access-management/users/) with an [API key](https://deploy.equinix.com/developers/docs/metal/identity-access-management/api-keys/).

If you have the [Metal CLI](https://deploy.equinix.com/developers/docs/metal/libraries/cli/) configured, the following will setup your authentication and project settings in an OSX or Linux shell environment.

```sh
eval $(metal env -o terraform --export) #
export TF_VAR_metal_metro=sl # Deploy to Seoul
```

Otherwise, copy `terraform.tfvars.example` to `terraform.tfvars` and edit the input values before continuing.

Run the following from your console terminal:

```sh
terraform init
terraform apply
```

When complete, after roughly 45m, you'll see something like the following:

```console
Outputs:

bastion_public_ip = "ipv4-address"
nutanix_sos_hostname = [
  "uuid1@sos.sl1.platformequinix.com",
  "uuid2@sos.sl1.platformequinix.com",
  "uuid3@sos.sl1.platformequinix.com",
]
ssh_private_key = "/terraform/workspace/ssh-key-abc123"
ssh_forward_command = "ssh -L 9440:1.2.3.4:9440 -i /terraform/workspace/ssh-key-abc123 root@ipv4-address"
```

See ["Known Problems"](#known-problems) if your `terraform apply` does not complete successfully.

## Next Steps

You have several ways to access the bastion node, Nutanix nodes, and the cluster.

### Login to Prism GUI

- First create an SSH port forward session with the bastion host:

  **Mac or Linux**

  ```sh
  $(terraform output -raw ssh_forward_command)
  ```

  **Windows**

  ```sh
  invoke-expression $(terraform output -raw ssh_forward_command)
  ```

- Then open a browser and navigate to <https://localhost:9440>
- See [Logging Into Prism Central](https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Guide-vpc_2023_4:mul-login-pc-t.html) for more details (including default credentials)

### Access the Bastion host over SSH

For access to the bastion node, for troubleshooting the installation or to for network access to the Nutanix nodes, you can SSH into the bastion host using:

```sh
ssh -i $(terraform output -raw ssh_private_key) root@$(terraform output -raw bastion_public_ip)
```

### Access the Nutanix nodes over SSH

You can open a direct SSH session to the Nutanix nodes using the bastion host as a jumpbox. Debug details from the Cluster install can be found within `/home/nutanix/data/logs`.

```sh
ssh -i $(terraform output -raw ssh_private_key) -j root@$(terraform output -raw bastion_public_ip) nutanix@$(terraform output -raw cvim_ip_address)
```

### Access the Nutanix nodes out-of-band

You can access use the [SOS (Serial-Over-SSH)](https://deploy.equinix.com/developers/docs/metal/resilience-recovery/serial-over-ssh/) interface for [out-of-bands access using the default credentials for Nutanix nodes](https://deploy.equinix.com/developers/docs/metal/operating-systems/licensed/#accessing-your-nutanix-server).

```sh
ssh -i $(terraform output -raw ssh_private_key) $(terraform output -raw nutanix_sos_hostname[0]) # access the first node
ssh -i $(terraform output -raw ssh_private_key) $(terraform output -raw nutanix_sos_hostname[1]) # access the second node
ssh -i $(terraform output -raw ssh_private_key) $(terraform output -raw nutanix_sos_hostname[2]) # access the third node
```

## Known Problems

### On-Demand Instances

#### TODO

### SSH failures while running on macOS

The Nutanix devices have `sshd` configured with `MaxSessions 1`. In most cases this is not a problem, but in our testing on macOS we observed frequent SSH connection errors. These connection errors can be resolved by turning off the SSH agent in your terminal before running `terraform apply`. To turn off your SSH agent in a macOS terminal, run `unset SSH_AUTH_SOCK`.

### Other Timeouts and Connection issues

This POC project has not ironed out all potential networking and provisioning timing hiccups that can occur. In many situations, running `terraform apply` again will progress the deployment to the next step. If you do not see progress after 3 attempts, open an issue on GitHub: <https://github.com/equinix-labs/terraform-equinix-metal-nutanix-cluster/issues/new>.

## Examples

To view examples for how you can leverage this module, please see the [examples](examples/) directory.

<!-- TEMPLATE: The following block has been generated by terraform-docs util: https://github.com/terraform-docs/terraform-docs -->
<!-- BEGIN_TF_DOCS -->

## Requirements

| Name                                                                     | Version |
| ------------------------------------------------------------------------ | ------- |
| <a name="requirement_terraform"></a> [terraform](#requirement_terraform) | >= 1.0  |
| <a name="requirement_equinix"></a> [equinix](#requirement_equinix)       | >= 1.30 |
| <a name="requirement_null"></a> [null](#requirement_null)                | >= 3    |
| <a name="requirement_random"></a> [random](#requirement_random)          | >= 3    |

## Providers

| Name                                                               | Version |
| ------------------------------------------------------------------ | ------- |
| <a name="provider_equinix"></a> [equinix](#provider_equinix)       | >= 1.30 |
| <a name="provider_null"></a> [null](#provider_null)                | >= 3    |
| <a name="provider_random"></a> [random](#provider_random)          | >= 3    |
| <a name="provider_terraform"></a> [terraform](#provider_terraform) | n/a     |

## Modules

| Name                                         | Source         | Version |
| -------------------------------------------- | -------------- | ------- |
| <a name="module_ssh"></a> [ssh](#module_ssh) | ./modules/ssh/ | n/a     |

## Resources

| Name                                                                                                                                             | Type        |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- |
| [equinix_metal_device.bastion](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_device)                       | resource    |
| [equinix_metal_device.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_device)                       | resource    |
| [equinix_metal_gateway.gateway](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_gateway)                     | resource    |
| [equinix_metal_port.bastion_bond0](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_port)                     | resource    |
| [equinix_metal_port.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_port)                           | resource    |
| [equinix_metal_project.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_project)                     | resource    |
| [equinix_metal_reserved_ip_block.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_reserved_ip_block) | resource    |
| [equinix_metal_vlan.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_vlan)                           | resource    |
| [equinix_metal_vrf.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/resources/metal_vrf)                             | resource    |
| [null_resource.finalize_cluster](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource)                          | resource    |
| [null_resource.reboot_nutanix](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource)                            | resource    |
| [null_resource.wait_for_dhcp](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource)                             | resource    |
| [null_resource.wait_for_firstboot](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource)                        | resource    |
| [random_string.vrf_name_suffix](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string)                           | resource    |
| [terraform_data.input_validation](https://registry.terraform.io/providers/hashicorp/terraform/latest/docs/resources/data)                        | resource    |
| [equinix_metal_project.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/data-sources/metal_project)                  | data source |
| [equinix_metal_vlan.nutanix](https://registry.terraform.io/providers/equinix/equinix/latest/docs/data-sources/metal_vlan)                        | data source |

## Inputs

| Name                                                                                                   | Description                                                                                                                                                                                                                                                                                                                              | Type           | Default          | Required |
| ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | ---------------- | :------: |
| <a name="input_metal_auth_token"></a> [metal_auth_token](#input_metal_auth_token)                      | Equinix Metal API token.                                                                                                                                                                                                                                                                                                                 | `string`       | n/a              |   yes    |
| <a name="input_metal_metro"></a> [metal_metro](#input_metal_metro)                                     | The metro to create the cluster in.                                                                                                                                                                                                                                                                                                      | `string`       | n/a              |   yes    |
| <a name="input_metal_organization_id"></a> [metal_organization_id](#input_metal_organization_id)       | The ID of the Metal organization in which to create the project if `create_project` is true.                                                                                                                                                                                                                                             | `string`       | n/a              |   yes    |
| <a name="input_create_project"></a> [create_project](#input_create_project)                            | (Optional) to use an existing project matching `metal_project_name`, set this to false.                                                                                                                                                                                                                                                  | `bool`         | `true`           |    no    |
| <a name="input_create_vlan"></a> [create_vlan](#input_create_vlan)                                     | Whether to create a new VLAN for this project.                                                                                                                                                                                                                                                                                           | `bool`         | `true`           |    no    |
| <a name="input_metal_bastion_plan"></a> [metal_bastion_plan](#input_metal_bastion_plan)                | Which plan to use for the bastion host.                                                                                                                                                                                                                                                                                                  | `string`       | `"c3.small.x86"` |    no    |
| <a name="input_metal_project_id"></a> [metal_project_id](#input_metal_project_id)                      | The ID of the Metal project in which to deploy to cluster. If `create_project` is false and<br> you do not specify a project name, the project will be looked up by ID. One (and only one) of<br> `metal_project_name` or `metal_project_id` is required or `metal_project_id` must be set.                                              | `string`       | `""`             |    no    |
| <a name="input_metal_project_name"></a> [metal_project_name](#input_metal_project_name)                | The name of the Metal project in which to deploy the cluster. If `create_project` is false and<br> you do not specify a project ID, the project will be looked up by name. One (and only one) of<br> `metal_project_name` or `metal_project_id` is required or `metal_project_id` must be set.<br> Required if `create_project` is true. | `string`       | `""`             |    no    |
| <a name="input_metal_vlan_description"></a> [metal_vlan_description](#input_metal_vlan_description)    | Description to add to created VLAN.                                                                                                                                                                                                                                                                                                      | `string`       | `"ntnx-demo"`    |    no    |
| <a name="input_metal_vlan_id"></a> [metal_vlan_id](#input_metal_vlan_id)                               | ID of the VLAN you wish to use.                                                                                                                                                                                                                                                                                                          | `number`       | `null`           |    no    |
| <a name="input_nutanix_node_count"></a> [nutanix_node_count](#input_nutanix_node_count)                | The number of Nutanix nodes to create.                                                                                                                                                                                                                                                                                                   | `number`       | `3`              |    no    |
| <a name="input_nutanix_reservation_ids"></a> [nutanix_reservation_ids](#input_nutanix_reservation_ids) | Hardware reservation IDs to use for the Nutanix nodes. If specified, the length of this list must<br> be the same as `nutanix_node_count`. Each item can be a reservation UUID or `next-available`. If<br> you use reservation UUIDs, make sure that they are in the same metro specified in `metal_metro`.                              | `list(string)` | `[]`             |    no    |
| <a name="input_skip_cluster_creation"></a> [skip_cluster_creation](#input_skip_cluster_creation)       | Skip the creation of the Nutanix cluster.                                                                                                                                                                                                                                                                                                | `bool`         | `false`          |    no    |

## Outputs

| Name                                                                                            | Description                               |
| ----------------------------------------------------------------------------------------------- | ----------------------------------------- |
| <a name="output_bastion_public_ip"></a> [bastion_public_ip](#output_bastion_public_ip)          | The public IP address of the bastion host |
| <a name="output_nutanix_sos_hostname"></a> [nutanix_sos_hostname](#output_nutanix_sos_hostname) | The SOS address to the nutanix machine.   |
| <a name="output_ssh_private_key"></a> [ssh_private_key](#output_ssh_private_key)                | The private key for the SSH keypair       |

<!-- END_TF_DOCS -->

## Contributing

If you would like to contribute to this module, see [CONTRIBUTING](CONTRIBUTING.md) page.

## License

Apache License, Version 2.0. See [LICENSE](LICENSE).
