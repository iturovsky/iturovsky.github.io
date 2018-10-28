---
published: false
---
## Installing MS Exchange Server 2016 with Ansible

This post is the first one in series of post devoteed to Ansible + Windows. To say more precisely, not just Windows but Ansible + enterpise Apps like MS Exchange Server, SQL Server, etc. 

Ansible is shipped with modules for confiruing different core parts of Windows (files, registry, etc.), some Windows features like IIS but there is no say modules for managing MS Exchange or MS SQL (though there is one module for creation SQL databases). 

This post does not pretend to be best practice on how you would configure MS Exchange or other app, moreover for the sake of simplicity I will not sometimes follow best practices (like creating roles instead of keeping all the tasks in playbook), however it will demonstrate some techniques that can be used in the real life.

Thats being said, let's get started. 
## Environment and Ansible Inventory
In our scenario we will use 3 virtual Windows 2012 R2 machines:
- DC-1: this is our domain controller
- Exch2016-1: this will be the first server we will install Exchange on, currently it is just domain member
- Exch2016-2: this will be the second machine we will install Exchange on, currently it is just domain
  member

We need to machines for Exchange so we can configure DAG later.
Since this post is about Exchange, we will not cover AD forest/domain creation and process of joining servers to the domain though Ansible have appropriate modules ([win_domain](https://docs.ansible.com/ansible/latest/modules/win_domain_module.html#win-domain-module) and [win_domain_computer](https://docs.ansible.com/ansible/latest/modules/win_domain_computer_module.html#win-domain-computer-module) respecitvely).

There are a lot of guides on how to configure Windows and Ansible. Here are some links to official documentation which will help to decrypt inventory file below if required
- [Setting up a Windows Host](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html)
- [Windows Remote management](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html)

```ini
[exchange]
Exch2016-1.test.local
Exch2016-2.test.local

[ad]
DC-1.test.local

[win:children]
exchange
ad

[win:vars]
ansible_connection=winrm
ansible_winrm_transport=kerberos
ansible_psrp_auth=kerberos
ansible_winrm_server_cert_validation=ignore
```


## MS Exchange installation process
In order to create playbook we apparently need to understand what steps we need to perform to install MS Exchange Server 2016. 

These steps as well as system requirements are outlined in the following  Technet [section](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/plan-and-deploy?view=exchserver-2016) and generally include the following steps:
- AD changes:
    -  Instllation of prerequisites onn Domain Controller if we want to use domain controller to prepare AD (in this case we want)
    - Schema expansion 
    - AD preparation 
    - Individual domains preparation
- Exchange Server installation
  - Installation of prerequisites
  - Exchange setup
  

## AD changes
Now we will start compilint our playbook which will make required changes on AD side:
```ansible
---
- name: Prepare AD
  hosts: ad
  vars:
    pattern: '\d\d\d\d\d'
  tasks: 
```

### Prerequsites installation
Per [Exchange Server prerequisites](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/prerequisites?view=exchserver-2016#exchange-2016-prerequisites-for-preparing-active-directory) we nned to install the folowing components on the domain controller:
- .NET Framework 4.7.1
- Visual C++ Redistributable Packages for Visual Studio 2013
- RSAT-ADDS

####Installing .Net Framework
The problem with .Net installation is that it is not ordinary msi package which instllation state, version and productid can be easily checked in registry (or in Add/Remove programs applet in Control Panel) and thus we can't just use win_package module to install it. First, we need to determine which version of Framework is already installed. 

Per [How to: Determine which .NET Framework versions are installed](https://docs.microsoft.com/en-us/dotnet/framework/migration-guide/how-to-determine-which-versions-are-installed) we need to check the registry value _Release_ under _HKEYLOCALMACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full_
Luckely, Ansible has module [win_reg_stat](https://docs.ansible.com/ansible/latest/modules/win_reg_stat_module.html) created exactly for this purpose so we add the following task in our playbook:
```yml
  - name: Check .Net version
    win_reg_stat:
        path: HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full\
        name: Release
    register: netversion
```
If we execute this task with Ansible, we will receive the following result:

Ok, so now we know .Net version installed on our machine.  The next step will  be to install min required version if it is not yet installed. We will use the following task to accomplish it:

```yml
  - name: Install .NET Framework 4.7.1
    win_package:
      path: C:\xfer\NDP471-KB4033342-x86-x64-AllOS-ENU.exe
      product_id: '{0A0CADCF-78DA-33C4-A350-CD51849B9702}'
      arguments: /q /norestart
      state: present
    when: netversion.value < 461310
    register: dotnet_installation
```

As you see we used conditonal when - the task will only run when the netversion.value is lower than 461310 which is .NET Framework 4.7.1. 

Note  that we used /norestart arguement. This way we do not allow automatic reboot to happen after installation of .net since we do not want the playbook to fail when the machine is rebooted unexpectedly for Ansible. But we still need to reboot and therefore we will use win_reboot module for this purpose:
```yml
  - name: Reboot
    win_reboot:
      post_reboot_delay: 180
    when: (dotnet_installation.reboot_required | default(false))
```
A  couple of words about this when condition. As you see we  reboot the host depending on value of dotnet_installation variable which is registered when previous Install .NET Framework 4.7.1 is executed. What if the .Net installation task is skipped since when condition is false .i.e required version of .net is already installed? In this case dotnet_installation varialbe will not be defined and our current task will fail since we use undefined varible.  In order to avoid this failure, we set false value in when conditional. This will work the same way ISNULL works in SQL:  if dotnet_installtion.reboot_required has some value it will be used, if not the false value will be used in conditional. 

So we have .Net installed. Now it is time to take care about  Visual C++ Redistributable Packages for Visual Studio 2013. It is easy enough, since we can use built in win_package module:
```yml
  - name: Install Visual C++ Redistributable Packages for Visual Studio 2013
    win_package:
      arguments: /q /norestart
      product_id: '{929FBD26-9020-399B-9A7A-751D61F0B942}'
      state: present
      path: C:\xfer\vcredist_x64.exe
```

