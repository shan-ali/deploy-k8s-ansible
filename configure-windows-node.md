# Configure Your Local Windows Host as a Node

Normally ansible uses SSH to communicate with hosts, however, for windows we need to use WinRM. We will be follwing the steps outlined in [Official Windows Setup](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html)

Since we are on Windows 10, we can skip the initial steps of upgrading the powershell and .NET Framework versions. If you are on an older version of windows please follow the steps in [upgrading-powershell-and-net-framework](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#upgrading-powershell-and-net-framework).

## Create Ansible User

You will need to manually create an `ansible` windows user on your system that is part of the `Administrators` group. 

You can follow the step in this video: [Configure a Windows Host for Ansible - ansible winrm](https://www.youtube.com/watch?v=-vPXS8UuJoI&ab_channel=AnsiblePilot)

## WinRM Setup

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

>Note: credential for ansible user will be added later on as an object in AWX

![image](https://user-images.githubusercontent.com/16169323/162782967-3372148f-ca4e-4a2b-851d-3167365db1ef.png)

```
ansible_connection: winrm
ansible_winrm_transport: basic
ansible_winrm_server_cert_validation: ignore
```

## Add Windows Ansible User Credentials

Create a new [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `Windows Ansible User`

![image](https://user-images.githubusercontent.com/16169323/162782640-ebccbaf4-69b3-4a95-b857-18a3f2e3f98e.png)





References:
- https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html 
- https://www.youtube.com/watch?v=-vPXS8UuJoI&ab_channel=AnsiblePilot

## Test Windows Host (Optional)

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `Windows Test`. This project will use this git repository as its source

![image](https://user-images.githubusercontent.com/16169323/162069179-a40eb978-8e68-4bc1-b610-675a7868cdcb.png)

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) named `Hello World` that is in the project `Windows Test`. You will use the Playbook `helloworld.yml` that is in the root of this repository. 

![image](https://user-images.githubusercontent.com/16169323/162069250-243d323b-0141-4186-a14f-cd21cf3d415b.png)

Launch the job

![image](https://user-images.githubusercontent.com/16169323/162069552-a6d32138-9d14-476c-a627-05633bf6ddd9.png)

## Provision Multipass VMs using Ansible




