# install-awx-ansible-k8s

Deploy a K8s cluster on Multipass VMS using Ansible and AWX (Ansible Tower)

## Requirements

- [Multipass](https://multipass.run/docs/installing-on-windows) (Hyper-V)

## Technologies

- [AWX](https://github.com/ansible/awx/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [Docker](https://docs.docker.com/)
- [Kubernetes](https://kubernetes.io/docs/home/)
- [Ansible](https://docs.ansible.com/ansible/latest/index.html)

## AWX Multipass VM Setup

Multipass takes advantage of [cloud-init](https://ubuntu.com/blog/using-cloud-init-with-multipass) yaml files to customize hosts on launch. We will use one of these files when launching our vm for awx. `multipass/cloud-init/awx-cloud-config.yml` does the following: 

1. installs required packages
2. installs docker 20.10.12 as outlined in the [Offical Docker installation documention](https://docs.docker.com/engine/install/ubuntu/)
3. installs [minikube](https://minikube.sigs.k8s.io/docs/start/)
4. installs [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)
5. sets the cgroup driver to systemd for docker, by default it is cgroupfs

```
multipass launch --cloud-init multipass/cloud-init/awx-cloud-config.yml --disk 15G --mem 5G --cpus 4 --name awx
```

This will create a multipass vm named `awx`

## Disable Dynamic Memory Allocation

>Note: If you have a lot of memory (32G+) on your system you can probably skip this step. Otherwise you may run into out of memory errors for awx

When running on Hyper-V, Multipass will dynamically allocate more memory even if we specify --mem 3G. In order to disable the dynamic memory allocation we will need to

1. Stop the vm 
```
multipass stop awx
```
2. Disable dynamic memory allocation in Hyper-V

![image](https://user-images.githubusercontent.com/16169323/163237066-633345ec-5939-4763-bc6f-0434ed67510d.png)

3. Restart the vm
```
multipass start awx
```

## AWX Minikube Setup

The recommended way of running awx is on a minikube (or any k8s) instance. We will be installing and starting minikube on our awx vm.

Login to the newly created `awx` vm

```
multipass shell awx
```

Start minikube

```
minikube start --memory=4g --cpus=4
```
>Note: Using lower mem/cpu requirements may cause issues when starting awx pods

Set alias for minikube kubectl command to kubectl

```
alias kubectl="minikube kubectl --"
```

References: 
- https://minikube.sigs.k8s.io/docs/start/
- https://github.com/ansible/awx/blob/devel/INSTALL.md

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
You can now access the AWX webpage by going to [http://awx.mshome.net:8080/](http://awx.mshome.net:8080/) or `<awx-vm-ip-address>:8080`

>Note: you can find your ip address with `multipass list` or `ip a` in your vms shell

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

Once you have successfully completed this, continue from [Add New VMs to AWX](#add-new-vms-to-awx)

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

Create a new [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `kubernetes-cluster`

Add our `controller` and `worker` VMs as Host(s) to AWX. Make sure they are part of the `kubernetes-cluster` Inventory. 

![image](https://user-images.githubusercontent.com/16169323/163254955-0f275d61-3ed3-4b20-81d3-c535b4002118.png)

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

Create a new [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `ssh-awx` with user `ubuntu`. Paste the content of your private key in the `SSH Private Key` section.

![image](https://user-images.githubusercontent.com/16169323/163255423-94a07527-ce33-481d-b9fa-c87c7f8042e6.png)

## Install Packages

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `kubernetes-cluster` if you have not already. This project will use this git repository as its source.

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) with the following:

  1. Name is `package-install` 
  2. Inventory is `kubernetes-cluster`
  4. Project is `kubernetes-cluster`
  5. Playbook is [kubernetes/kubernetes-packages-install.yml](kubernetes/kubernetes-packages-install.yml). This playbook: 
     - Installs Docker (docker-ce, docker-ce-cli, containerd.io) and required packages and sets configurations
     - Installs Kubernetes (kubelet, kubeadmn, kubectl) and sets configurations  
  7. Credentials are `ssh-awx`

![image](https://user-images.githubusercontent.com/16169323/163255763-f2928f92-8114-4495-8b23-134b9ad13858.png)

Launch the job. If everything is successful, your VMs will now have Docker and Kuberenetes installed. 

![image](https://user-images.githubusercontent.com/16169323/163260704-1ab5d9f5-e764-4297-8348-f438d747b780.png)

## Initialize the Kubernetes Cluster

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) with the following:

  1. Name is `kubernetes-init` 
  2. Inventory is `kubernetes-cluster`
  4. Project is `kubernetes-cluster`
  5. Playbook is [/ansible/kubernetes-init.yml](/ansible/kubernetes-init.yml). This playbook: 
     - runs `kubeadm reset` to clean up any existing cluster & cleans existing kube config
     - runs `kubeadm init` on `controller` & sets up kube config for ubuntu user
     - installs calico cni for pod networking
     - runs `kubeadm join` on `worker`
  7. Credentials are `ssh-awx`

![image](https://user-images.githubusercontent.com/16169323/163256638-9d46e917-d17f-45d9-bda8-dec0eaa8e8ed.png)

Launch the job. If everything is successful, you will now have a kubernetes cluster with one control plane and one worker node!

![image](https://user-images.githubusercontent.com/16169323/163260608-02ade009-3a2c-483b-93f5-e054cfbe7040.png)


## Verify Cluster

on `controller` vm

```
kubectl get nodes
```
>output
```
NAME         STATUS   ROLES                  AGE     VERSION
controller   Ready    control-plane,master   6m11s   v1.23.5
worker       Ready    <none>                 5m18s   v1.23.5
```

## Smoke Test

on `controller` vm

### Deployments Test

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l app=nginx
```

> output

```
NAME                    READY   STATUS    RESTARTS   AGE
nginx-dbddb74b8-6lxg2   1/1     Running   0          10s
```

### Services Test

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Create a service to expose deployment nginx on node ports.

```
kubectl expose deploy nginx --type=NodePort --port 80
```


```
PORT_NUMBER=$(kubectl get svc -l app=nginx -o jsonpath="{.items[0].spec.ports[0].nodePort}")
```

Test to view NGINX page

```
curl http://worker-1:$PORT_NUMBER
curl http://worker-2:$PORT_NUMBER
```

> output

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```





