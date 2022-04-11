# Provision Multipass VMs using Ansible

At this point you should have the following in you AWX environment related to your Windows host:

1. An [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `Windows Local`
2. A `Host` using your Window's host's IP address
3. [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `Windows Ansible User`

## Create K8s Cluster Project 

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `K8s Cluster`. This project will use this git repository as its source.

## Create Multipass Provisioning Job

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) named `Multipass Provision VMs` that is in the project `K8s Cluster`. You will use the Playbook [multipass-provision-vms.yml](/multipass/multipass-provision-vms.yml) that is in the `multipass` directory. 





