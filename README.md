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
cd awx/awx-operator
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

### Install AWX

```
cd ../awx-main
kustomize build . | kubectl apply -f -
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

You can now access the AWX webpage by going to `localhost:30080`. Port is from the awx-service NodePort from above. 

By default, the admin user is `admin` and the password is available in the `<resourcename>-admin-password` secret. To retrieve the admin password, run:

```
$awxAdminPassword=$(kubectl get secret awx-admin-password -o jsonpath="{.data.password}" -n awx)
[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($awxAdminPassword))
```

Reference:
- https://github.com/ansible/awx/blob/devel/INSTALL.md
- https://github.com/ansible/awx-operator 
