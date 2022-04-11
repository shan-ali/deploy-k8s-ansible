# Provision Multipass VMs using Ansible

At this point you should have the following in you AWX environment related to your Windows host:

1. An [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `Windows Local`
2. A `Host` using your Window's host's IP address
3. [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `Windows Ansible User`

## Create K8s Cluster Project 

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `K8s Cluster`. This project will use this git repository as its source.

## Create Multipass Provisioning Job

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) with the following:

  1. Name is `Multipass Provision VMs` 
  2. Inventory is `Windows Local`
  4. Project is `K8s Cluster`
  5. Playbook is [multipass/multipass-provision-vms.yml](/multipass/multipass-provision-vms.yml)
  6. Credentials are `Windows Ansible User`

![image](https://user-images.githubusercontent.com/16169323/162786182-0d563e31-3b71-4275-973a-457722e5443a.png)

## Launch Multipass Provision VMs Job


