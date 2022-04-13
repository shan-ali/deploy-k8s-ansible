# Provision Multipass VMs using Ansible

At this point you should have the following in you AWX environment related to your Windows host:

1. An [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `windows-local`
2. A `Host` using your Window's host's IP address
3. [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `windows-ansible-user`

## Create Kubernetes Cluster Project 

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `kubernetes-cluster`. This project will use this git repository as its source.

## Create Multipass Provisioning Job

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) with the following:

  1. Name is `provision-vms` 
  2. Inventory is `windows-local`
  4. Project is `kubernetes-cluster`
  5. Playbook is [ansible/provision-vms.yml](ansible/provision-vms.yml)
  6. Credentials are `windows-ansible-user`

![image](https://user-images.githubusercontent.com/16169323/163248360-6cf955a7-ec40-4f43-a2d7-ab990aa87f38.png)

## Launch Multipass Provision VMs Job

The `provision-vms.yml` playbook will essentially create two Multipass VMs. It will also apply netplan configurations to create static IP addresses for each VM. 
1. controller - 172.22.5.11/20
2. worker - 172.22.5.21/20

![image](https://user-images.githubusercontent.com/16169323/163254391-0267921c-52c7-4bd8-a94e-bdd068c54796.png)


