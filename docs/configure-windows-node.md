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

Create a new [Inventory](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_inventory.html) named `windows-local`

![image](https://user-images.githubusercontent.com/16169323/163241541-9feea6a4-8af7-4318-ac6c-a8a2305f8c5a.png)

Create a new `Host` using your Window's Host's IP address. You will need to add the [basic authentication](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#basic) varibles using your newly created ansible user. 

>Note: credential for ansible user will be added later on as an object in AWX

![image](https://user-images.githubusercontent.com/16169323/163241693-bf57fc42-11ae-4ba4-b513-50a7216d757e.png)

```
ansible_connection: winrm
ansible_winrm_transport: basic
ansible_winrm_server_cert_validation: ignore
```

## Add Windows Ansible User Credentials

Create a new [Credential](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_credential.html) named `windows-ansible-user`. 

Any [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) that will run on your Windows local will need to use these credentials

![image](https://user-images.githubusercontent.com/16169323/163242648-618c643c-a85d-4646-b49d-5d6909ff7964.png)

## Test Windows Host (Optional)

Create a new [Project](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_project.html) named `windows-test`. This project will use this git repository as its source

![image](https://user-images.githubusercontent.com/16169323/163242119-2b8b350f-1c84-41c2-8b93-70feb706cf0b.png)

Create a new [Job Template](https://docs.ansible.com/ansible-tower/latest/html/quickstart/create_job.html) with the following:

  1. Name is `hello-world-windows` 
  2. Inventory is `windows-local`
  4. Project is `windows-test`
  5. Playbook is [ansible/helloworld_win.yml](ansible/helloworld_win.yml). This playbook: 
  7. Credentials are `windows-ansible-user`

![image](https://user-images.githubusercontent.com/16169323/163242385-7ce014c5-8ef4-434e-9936-4290a17dbb4d.png)

Launch the job

![Uploading image.png…]()



