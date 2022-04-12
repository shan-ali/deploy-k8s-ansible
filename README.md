# install-awx-ansible-k8s

Deploy a K8s cluster on Multipass VMS using Ansible and AWX (Ansible Tower)

## Requirements

- [Multipass](https://multipass.run/docs/installing-on-windows)

## Technologies

- [AWX](https://github.com/ansible/awx/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [Docker](https://docs.docker.com/)
- [Kubernetes](https://kubernetes.io/docs/home/)
- [Ansible](https://docs.ansible.com/ansible/latest/index.html)

## Multipass Setup

Multipass takes advantage of [cloud-init](https://ubuntu.com/blog/using-cloud-init-with-multipass) yaml files to customize hosts on launch. In the `awx/multipass` directory there is a `awx-cloud-config.yml` file that does the following when provisioning the vm. 

1. installs required packages
2. installs docker 20.10.12 as outlined in the [Offical Docker installation documention](https://docs.docker.com/engine/install/ubuntu/)
3. installs [minikube](https://minikube.sigs.k8s.io/docs/start/)
4. installs [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)
5. sets the cgroup driver to systemd for docker, by default it is cgroupfs

```
multipass launch --cloud-init awx/multipass/awx-cloud-config.yml --disk 15G --mem 4G --cpus 4 --name awx
```

This will create a multipass vm named `awx`

## Minikube Setup

Login to the newly created `awx` vm

```
multipass shell awx
```

Start minikube

```
minikube start --memory=3g --cpus=4
```
>Note: Using lower mem/cpu requirements may cause issues when starting awx pods

Set alias for minikube kubectl command to kubectl

```
alias kubectl="minikube kubectl --"
```

References: https://minikube.sigs.k8s.io/docs/start/

## AWX Setup 

### Clone Git Repository

```
git clone https://github.com/shan-ali/install-awx-ansible-k8s
cd install-awx-ansible-k8s
```
### Install AWX Operator

```
kustomize build ./awx/awx-operator/ | kubectl apply -f -
```

Make sure `awx-operator` is running
```
kubectl get pods -n awx
```
>output 
```
NAME                                               READY   STATUS    RESTARTS   AGE
awx-operator-controller-manager-557589c5f4-ck5t6   2/2     Running   0          60s
```

### Install AWX

```
kustomize build ./awx/awx-main/ | kubectl apply -f -
```

View logs

```
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager -n awx
```
> Note: `-n awx` is stating to use the k8s namespace awx

View pods created by awx operator

```
kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator" -n awx
```
> output
```
NAME                   READY   STATUS    RESTARTS   AGE
awx-74bdc7c78d-g4spr   4/4     Running   0          88s
awx-postgres-0         1/1     Running   0          2m25s
```

View services created by awx operator

```
kubectl get svc -l "app.kubernetes.io/managed-by=awx-operator" -n awx
```
> output
```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
awx-postgres   ClusterIP   None            <none>        5432/TCP       4m11s
awx-service    NodePort    10.107.73.105   <none>        80:30080/TCP   3m18s
```
### Access AWX

Expose Kubernetes port for external accesss

```
kubectl port-forward --address 0.0.0.0 service/awx-service 8080:80 -n awx &> /dev/null &
```
You can now access the AWX webpage by going to `<multipass-awx-vm-ip-address>:8080` or `http://awx.mshome.net:8080/`

>Note: you can find your ip address with `multipass list`

By default, the admin user is `admin` and the password is available in the `<resourcename>-admin-password` secret. To retrieve the admin password, run:

```
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" -n awx| base64 --decode
```

Reference:
- https://github.com/ansible/awx/blob/devel/INSTALL.md
- https://github.com/ansible/awx-operator 

## Configure Your Local Windows Host as a Node (Optional)

Now that we have AWX up and running, we can start configuring it and adding nodes to it. The next main step will be to setup and run a playbook to deploy a K8s cluster on two Multipass VMs. 

If you would like to automate the provisoning of these VMs, you will need to add your Windows local machine as a node to AWX. Otherwise, you can provision the VMs with `multipass` commands manually by skipping to the [Provision VMs](#provision-vms) section. 

The steps to add your Windows machine as a node can be found here: [configure-windows-node.md](docs/configure-windows-node.md). 

## Provision VMs via Playbook (Optional)

If you have done the above step of configuring your Windows host as an Ansible node you can now provision your K8s Multipass VMs following the the steps in [provision-multipass-vms.md](docs/provision-multipass-vms.md). Otherwise skip to the [Provision VMs](#provision-vms) section. 

Once you have successfully completed this, continue from [Install Packages](#install-packages)

## Provision Kubernetes VMs

Change directory to `kubernetes/multipass`
```
cd kubernetes/multipass
```

Launch VMs
```
multipass launch --disk 5G --mem 1G --cpus 1 --name controller
multipass launch --disk 5G --mem 1G --cpus 1 --name worker
```

Copy netplan configuration

```
multipass transfer 01-controller-network.yaml controller:01-controller-network.yaml
multipass transfer 01-worker-network.yaml worker:01-worker-network.yaml
multipass exec controller -- sudo cp 01-controller-network.yaml /etc/netplan/ 
multipass exec worker -- sudo cp 01-worker-network.yaml /etc/netplan/ 
```

Restart VMs to apply netplan changes
```
multipass restart controller worker
```

Leave the directory
```
cd ../..
```
## Add New VMs to AWX

Create a new [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `K8s Cluster`

Add our `controller` and `worker` VMs as Host(s) to AWX. Make sure they are part of the `K8s Cluster` Inventory. 

![image](https://user-images.githubusercontent.com/16169323/162816434-57f54a41-2ad9-4b94-8cb2-947782f8ac38.png)

>Note: we can use the name of the VM or the IP address

## Setup SSH Keys

### Generate Keys

Login to the `awx` multipass instance if you are not already

```
multipass shell awx
```

Generate public and private keys. You can leave all setting as default. 
```
ssh-keygen -t rsa
```

At this point you should have 
- private key: `/home/ubuntu/.ssh/id_rsa`
- public key: `/home/ubuntu/.ssh/id_rsa.pub`

### Copy Public Key to Hosts

View the generated public key ID at:

```
cat /home/ubuntu/.ssh/id_rsa.pub
```
>output

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD......8+08b ubuntu@awx
```

Move public key of awx to `controller` and `worker` VMs. Access each VM using the following multipass command

```
multipass shell controller
```

Then add public key from awx to the authorized keys for all hosts

```
cat >> ~/.ssh/authorized_keys <<EOF
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD......8+08b ubuntu@awx
EOF
```

### Add Private Key as a Credential

Copy the output of the private key file

```
cat /home/ubuntu/.ssh/id_rsa
```

Create a new [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `SSH AWX` with user `ubuntu`. Paste the content of your private key in the `SSH Private Key` section.

![image](https://user-images.githubusercontent.com/16169323/162825145-a1f673a7-501f-42ca-b080-b668561ad4e5.png)

## Install Packages

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `K8s Cluster` if you have not already. This project will use this git repository as its source.

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) with the following:

  1. Name is `K8s Package Install` 
  2. Inventory is `K8s Cluster`
  4. Project is `K8s Cluster`
  5. Playbook is [kubernetes/kubernetes-packages-install.yml](kubernetes/kubernetes-packages-install.yml)
  6. Credentials are `SSH AWX`

![image](https://user-images.githubusercontent.com/16169323/162826951-1bc339e7-60de-4045-9b60-af792fb747f1.png)










