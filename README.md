<div align="center"><img src="./docs/images/panda.png" width="300"/></div>
<h2 align="center">🐼 ❤️.oO<br>"Pandas love everything"</h2>
<h1 align="center">Infrastructure code for Dev/Testnets</h1>

<p align="center">
<a href="https://github.com/ethpandaops/template-testnet/actions/workflows/ansible_lint.yaml"><img src="https://github.com/ethpandaops/template-testnet/actions/workflows/ansible_lint.yaml/badge.svg"></a>
<a href="https://github.com/ethpandaops/template-testnet/actions/workflows/terraform_lint.yaml"><img src="https://github.com/ethpandaops/template-testnet/actions/workflows/terraform_lint.yaml/badge.svg"></a>
<a href="https://github.com/ethpandaops/template-testnet/actions/workflows/helm_lint.yaml"><img src="https://github.com/ethpandaops/template-testnet/actions/workflows/helm_lint.yaml/badge.svg"></a>
</p>

This repository contains the infrastructure code used to setup ~all~ dev/testnets. A lot of the code uses reusable components either provided by our [ansible collection](https://github.com/ethpandaops/ansible-collection-general) or our [helm charts for kubernetes](https://github.com/ethpandaops/ethereum-helm-charts/).

# Networks

Status   | Network    | Links   | Ansible                                                      | Terraform | Kubernetes
------   | --------   | ----     |  -----                                                       | -------   | ------
 🟢Template🔴 | [devnet-0](https://template.devnet.io/)   | [Network config](network-configs/devnet-0) / [Inventory](https://bootnode-1.srv.template-testnet.ethpandaops.io/meta/api/v1/inventory.json)     | [🔗](ansible/inventories/devnet-0) | [🔗](terraform/devnet-0) | [🔗](kubernetes/devnet-0)

# Development
## Version management for tools

We're using [asdf](https://github.com/asdf-vm/asdf) to make sure that we all use the same versions across tools. Our repositories should contain versions defined in .tools-versions.

You can then use [`./asdf-setup.sh`](./asdf-setup.sh) to install all dependencies.

## Terraform
From [`./terraform/devnet-0/`](./terraform/devnet-0/)
```shell
terraform init
terraform apply
```

## Ansible
To install the nodes according to the inventory file that is generated by the terraform template run the following commands from [`./ansible/`](./ansible/)
```shell
./install_dependencies.sh
ansible-playbook -i inventories/devnet-0/inventory.ini playbook.yaml
```
In order to clean up the deployment
```shell
ansible-playbook -i inventories/devnet-0/inventory.ini cleanup_ethereum.yaml
```

# Spinning up a New Testnet
To create a new testnet using the infrastructure code, follow these steps:

## Validator Configuration
1. Open the [main.tf](terraform/devnet-0/main.tf) file located in the terraform/devnet-0/ directory.
2. Locate the sections that define the different nodes and their corresponding validator ranges, for this example this is `variable "digitalocean_vm_groups"`
3. Adjust the validator indexes in the terraform/main.tf file based on your desired allocation of validators to nodes.

For example, let's say you want to assign validator index 0-24 to the lodestar-besu-1 node and validator index 25-224 to the lighthouse-nethermind-1 node. Update the main.tf file as follows:
```terraform
    {
      id = "lodestar-besu"
      vms = {
        "1" = { ansible_vars : "validator_start=0 validator_end=25" }
      },
    },
    {
      id = "lighthouse-nethermind"
      vms = {
        "1" = { ansible_vars : "validator_start=25 validator_end=225" }
      },
    },
```
Make sure to adjust the validator ranges according to your requirements and the number of validators in your network. This configuration ensures that validators within the specified ranges will be allocated to the corresponding nodes during the deployment. By customizing the validator indexes in the Terraform configuration, you can allocate validators to specific nodes in your network according to your desired configuration.

4. `terraform apply` will create a the machines and the inventory file for you. The inventory file will be located in the [ansible/inventories/devnet-0](ansible/inventories/devnet-0/) directory.
5. The inventory.ini file will have the list of all the nodes that were created by Terraform. The inventory file will also have the validator ranges that were specified in the Terraform configuration. The validator ranges will be used by the Ansible playbook to allocate validators to the corresponding nodes.
```ini
[lodestar_besu]
lodestar-besu-1 ansible_host=167.99.34.241 cloud=digitalocean cloud_region=ams3 validator_start=0 validator_end=25
...
```
6. Adjust the total number of validators in the [ansible/inventories/devnet-0/group_vars/all.yaml](ansible/inventories/devnet-0/group_vars/all.yaml) file (ethereum_genesis_generator_config_files.values.env.NUMBER_OF_VALIDATORS) to match with your total number of validators that you are running.. This will be used by the Ansible playbook to generate the validator keys and deposit data for the network.

## Network Configuration
1. [ansible/inventories/devnet-0/group_vars/all.yaml](ansible/inventories/devnet-0/group_vars/all.yaml) has all the network configuration parameters. Adjust the parameters according to your requirements. Most likely you will not need to adjust these, unless you would like to use a custom setup. The default configuration will work for most networks.

## Deploying the Network
1. Run `ansible-playbook -i inventories/devnet-0/inventory.ini playbook.yaml` from the [ansible/](ansible/) directory to deploy the network. This will generate the genesis file, validators and deploy the network according to the configuration parameters specified in the [ansible/inventories/devnet-0/group_vars/all.yaml](ansible/inventories/devnet-0/group_vars/all.yaml) file. 

## Cleaning up the Network
1. Run `ansible-playbook -i inventories/devnet-0/inventory.ini cleanup_ethereum.yaml` from the [ansible/](ansible/) directory to clean up the network. This will delete the genesis file, validators and clean up the network on all the nodes.
2. Run `ansible-playbook -i inventories/devnet-0/inventory.ini ansible-playbook playbook.yaml -t ethereum_genesis -e ethereum_genesis_cleanup=true` from the [ansible/](ansible/) directory to clean up the network-configs and validators directories on your local machine. This step is required if you would like to reuse the nodes but with a different genesis configuration. (For example, if you would like to change the validator indexes assigned to the nodes, due to a relaunch).
3. Run `terraform destroy` from the [terraform/devnet-0/](terraform/devnet-0/) directory to delete the nodes. This will remove all the virtual machines and the inventory file. Be careful when running this command, as it will delete all the nodes and the inventory file. You will need to run `terraform apply` again to create the nodes and the inventory file.