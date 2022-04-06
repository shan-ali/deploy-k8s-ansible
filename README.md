# install-awx-ansible-k8s

Deploy a K8s cluster on Multipass VMS using Ansible and AWX (Ansible Tower)

## Requirements

This is all done on a Windows 10 machine

- [Docker Desktop](https://www.docker.com/products/docker-desktop) with Kubernetes enabled
- [Multipass](https://multipass.run/docs/installing-on-windows)
- [Chocolatey](https://chocolatey.org/install) (to install Kustomize)
- [Kustomize](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)

## Technologies

- [AWX](https://github.com/ansible/awx/)

## Install Kustomize

Open an cmd terminal as Administrator

```
choco install kustomize -y 
```

Reference: https://kubectl.docs.kubernetes.io/installation/kustomize/chocolatey/

## Install AWX on Docker Desktop K8s Cluster

### Install AWX Operator

```
cd awx
kustomize build . | kubectl apply -f -
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

Reference:
- https://github.com/ansible/awx/blob/devel/INSTALL.md
- https://github.com/ansible/awx-operator 
