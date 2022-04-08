# install-awx-ansible-k8s

Deploy a K8s cluster on Multipass VMS using Ansible and AWX (Ansible Tower)

## Requirements

- [Multipass](https://multipass.run/docs/installing-on-windows)

## Technologies

- [AWX](https://github.com/ansible/awx/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)

## Multipass Setup

Multipass takes advantage of [cloud-init](https://ubuntu.com/blog/using-cloud-init-with-multipass) yaml files to customize hosts on launch. In the `awx/multipass` directory there is a `awx-cloud-config.yml` file that does the following when provisioning the vm. 

1. installs required packages
2. installs docker 20.10.12 as outlined in the [Offical Docker installation documention](https://docs.docker.com/engine/install/ubuntu/)
3. installs [minikube](https://minikube.sigs.k8s.io/docs/start/)
4. installs [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)
5. sets the cgroup driver to systemd for docker, by default it is cgroupfs

```
multipass launch --cloud-init awx/multipass/awx-cloud-config.yml --disk 15G --mem 3g --cpus 4 --name awx
```

This will create a multipass vm named `awx`

## Minikube Setup

Login to the newly created `awx` vm

```
multipass shell awx
```

Start minikube

```
minikube start --memory=2g --cpus=4
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
You can now access the AWX webpage by going to `<multipass-awx-vm-ip-address>:8080`

>Note: you can find your ip address with `multipass list`

By default, the admin user is `admin` and the password is available in the `<resourcename>-admin-password` secret. To retrieve the admin password, run:

```
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" -n awx| base64 --decode
```

Reference:
- https://github.com/ansible/awx/blob/devel/INSTALL.md
- https://github.com/ansible/awx-operator 

## Configure Your Local Windows Host as a Node

Normally ansible uses SSH to communicate with hosts, however, for windows we need to use WinRM. We will be follwing the steps outlined in [Official Windows Setup](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html)

Since we are on Windows 10, we can skip the initial steps of upgrading the powershell and .NET Framework versions. If you are on an older version of windows please follow the steps in [upgrading-powershell-and-net-framework](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#upgrading-powershell-and-net-framework).

### Create Ansible User

You will need to manually create an `ansible` windows user on your system that is part of the `Administrators` group. 

You can follow the step in this video: [Configure a Windows Host for Ansible - ansible winrm](https://www.youtube.com/watch?v=-vPXS8UuJoI&ab_channel=AnsiblePilot)

### WinRM Setup

"There are two main components of the WinRM service that governs how Ansible can interface with the Windows host: the listener and the service configuration settings" 

The "script `ConfigureRemotingForAnsible.ps1` can be used to set up the basics. This script sets up both HTTP and HTTPS listeners with a self-signed certificate and enables the Basic authentication option on the service."

>Note: Open a powershell terminal as Administrator

```
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)

powershell.exe -ExecutionPolicy ByPass -File $file
```

## Add Windows Host to AWX

Create a new [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `Windows Local`

![image](https://user-images.githubusercontent.com/16169323/162064083-0a524e50-1699-4584-97b0-2bdea94c7cac.png)

Create a new `Host` using your Window's Host's IP address. You will need to add the [basic authentication](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#basic) varibles using your newly created ansible user.

![image](https://user-images.githubusercontent.com/16169323/162068763-404a51c3-1da5-4fed-88b4-50e92e9b5d7b.png)

```
ansible_user: ansible
ansible_password: <password>
ansible_connection: winrm
ansible_winrm_transport: basic
ansible_winrm_server_cert_validation: ignore
```

References:
- https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html 
- https://www.youtube.com/watch?v=-vPXS8UuJoI&ab_channel=AnsiblePilot

## Test Windows Host

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `Windows Test`. This project will use this git repository as its source

![image](https://user-images.githubusercontent.com/16169323/162069179-a40eb978-8e68-4bc1-b610-675a7868cdcb.png)

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) named `Hello World` that is in the project `Windows Test`. You will use the Playbook `helloworld.yml` that is in the root of this repository. 

![image](https://user-images.githubusercontent.com/16169323/162069250-243d323b-0141-4186-a14f-cd21cf3d415b.png)

Launch the job

![image](https://user-images.githubusercontent.com/16169323/162069552-a6d32138-9d14-476c-a627-05633bf6ddd9.png)

## Provision Multipass VMs using Ansible

