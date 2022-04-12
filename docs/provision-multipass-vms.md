# Provision Multipass VMs using Ansible

At this point you should have the following in you AWX environment related to your Windows host:

1. An [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `Windows Local`
2. A `Host` using your Window's host's IP address
3. [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `Windows Ansible User`

## Create K8s Cluster Project 

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `K8s Cluster`. This project will use this git repository as its source.

## Create Multipass Provisioning Job

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) with the following:

  1. Name is `Provision VMs` 
  2. Inventory is `Windows Local`
  4. Project is `K8s Cluster`
  5. Playbook is [kubernetes/multipass/kubernetes-provision-vms.yml](kubernetes/multipass/kubernetes-provision-vms.yml)
  6. Credentials are `Windows Ansible User`

![image](https://user-images.githubusercontent.com/16169323/163009577-e42a44a2-13a3-4e31-8ff9-362fbab73dbb.png)

## Launch Multipass Provision VMs Job

The `multipass-provision-vms.yml` playbook will essentially create two Multipass VMs. It will also apply netplan configurations to create static IP addresses for each VM. 
1. controller - 172.22.5.11/20
2. worker - 172.22.5.21/20

![image](https://user-images.githubusercontent.com/16169323/162809617-eeb5b3aa-91f6-400e-9746-b232e741a415.png)


