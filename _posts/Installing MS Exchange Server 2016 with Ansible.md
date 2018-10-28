---
published: false
---
## Installing MS Exchange Server 2016 with Ansible

This post is the first one in series of post devoteed to Ansible + Windows. To say more precisely, not just Windows but Ansible + enterpise Apps like MS Exchange Server, SQL Server, etc. 

Ansible is shipped with modules for confiruing different core parts of Windows (files, registry, etc.), some Windows features like IIS but there is no say modules for managing MS Exchange or MS SQL (though there is one module for creation SQL databases). 

This post does not pretend to be best practice on how you would configure MS Exchange or other app, moreover for the sake of simplicity I will not sometimes follow best practices insome cases (like creating roles instead of keeping all the tasks in monolith playbook), however it will demonstrate some techniques that can be used in the real life.

Thats being said, let's get started. 
## Environment and Ansible Inventory
In our scenario we will use 3 virtual Windows 2012 R2 machines:
- DC-1: this is our domain controller
- Exch2016-1: this will be the first server we will install Exchange on, currently it is just domain member
- Exch2016-2: this will be the second machine we will install Exchange on, currently it is just domain
  member as well

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
Per [Exchange Server prerequisites](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/prerequisites?view=exchserver-2016#exchange-2016-prerequisites-for-preparing-active-directory) we need to install the folowing components on the domain controller:
- .NET Framework 4.7.1
- Visual C++ Redistributable Packages for Visual Studio 2013
- RSAT-ADDS

####Installing .Net Framework
The problem with .Net installation is that it is not ordinary msi package which installation state, version and productid can be easily checked in registry (or in Add/Remove programs applet in Control Panel) and thus we can't just use _winpackage_ module to install it. First, we need to determine which version of Framework is already installed. 

Per [How to: Determine which .NET Framework versions are installed](https://docs.microsoft.com/en-us/dotnet/framework/migration-guide/how-to-determine-which-versions-are-installed) we can check the registry value _Release_ under _HKEYLOCALMACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full_
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

### Working with Active Directory
Now our domain controller is ready for AD changes. The first step is schema expansion
#### Expanding Schema
The schema can be expanded by running Setup.exe command:
```cmd
Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareSchema
```
This command behavior does not suit very vell for idempotancy purposes - even if the schema is already expanded, the second run of this command will still report that schema is successfully expanded. Thus we need to find the way to determine if the schema is already installed. 
The answer is [here](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/active-directory/ad-changes?view=exchserver-2016#extend-the-active-directory-schema):
> After the schema has been extended by running the /PrepareSchema command, the _/PrepareAD command, or installing the first Exchange server using the Exchange Setup wizard, the schema version is set in the ms-Exch-Schema-Version-Pt attribute. To verify that the Active Directory schema was extended successfully, you can check the value stored in this attribute

In addition, this [article](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/prepare-ad-and-domains?view=exchserver-2016#exchange-active-directory-versions) contains the values of this attribute. If we are installing CU10, it will be 15532. In order to check the attribute value, we will use Ansible's win_command module with good old _dsquery_.  

```yml
  - name: Get Schema Version
    win_command:  dsquery * CN=ms-Exch-Schema-Version-Pt,CN=Schema,CN=Configuration,DC=test,DC=local -attr rangeUpper
    register: rangeUpper
    changed_when: false
    failed_when: "'Directory object not found' not in Schema_version.stderr and Schema_version.stderr!=''"
    check_mode: no
```

Some comments on this task:
1. Since we use win_command module, Ansible will show this task as changed every time it runs. Since it is simple get task, we use _changedwhen:false_ to indicate that task does not change anything
2. Our next task (Schema expansion) will depend on result of this task. Thus in order to get this task executed even in check mode so our next task conditional could be evaluated, we use _checkmode:no_
3. As any other query dsquery can fail which will cause it to exit with some non-zero exit code and to record some error message in stderr. Using _failedwhen_ we tell Ansible to fail only when we receive smthing in stderr and this smth is not  'Directory object not found' message  which can be expected when schema is not yet expanded and the object CN=ms-Exch-Schema-Version-Pt,CN=Schema,CN=Configuration,DC=test,DC=local does not exist. 
Ok, let see what this task  will return
![schemacheck.PNG]({{site.baseurl}}/_posts/schemacheck.PNG)
As we see the required attribute value is returned as second element of stdout_lines dictionary. 
So we can grab it and covert to int the following way:
```yml
  - set_fact:
      Schema: "{{ rangeUpper.stdout_lines[1] | int }}"
 ```
 Now we have attribute's  value in Schema variable. Let's check if we need to expand schema and do expansion if required.
```yml
  - name: Extend Schema
    win_command: D:\Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareSchema
    when: Schema!=15332
    vars:
      ansible_become: yes
      ansible_become_method: runas
      ansible_become_user: TEST\Administrator
      ansible_become_pass: P@$$w0rd
    notify:
    - Pausing for replication to occur
```
This task will run if Schema is not equal 15332.
If we take a look on this task, we see that we have something in vars section of the task. 
Since the task of Schema expansion needs to be run as Schema Admin, we use separate account for this purpose with become.
Next,  we have _notify_. This directive will notify handler - it is special kind of task that runs at the end of playbook executinion only when a task changes state of the system and notifies handler about it. Classic example is to notify some service to restart when config is changed. 
In our case we will trigger handler if Schema is expanded. 
```yml
  handlers:
  - name: Pausing for replication to occur
    pause:
      minutes: 2
```
Handler just paused playbook execution for 2 minutes. There is also one nuance. As I said the handler will run at the end of playbook execution and we have to use special instruction flush_handlers  to run hanlder immidiately after task execution:
```yml
- meta: flush_handlers
```

Now it is time to prepare AD

####Preparing AD
AD can be prepared with command
```cmd
Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareAD  /OrganizationName:"Test"
```
But again for the purpose of keeping our playbook idempotent, we have to have separate task to find out if we need to PrepareAD. In order to do this we will use objectVersion attribute of Services>Microsoft Exchange>'Orgnization  Name' container.
```yml
  - name: Get Exchange System Object Version
    win_command:  dsquery * "CN=Test,CN=Microsoft Exchange,CN=Services,CN=Configuration,DC=test,DC=local" -attr objectVersion
    register: Object_version_configuration
    changed_when: false
    failed_when: "'Directory object not found' not in ExchObject_version.stderr and ExchObject_version.stderr!=''"
    check_mode: no

  - set_fact:
      ExchObj_config: "{{ Object_version_configuration.stdout_lines[1] | int }}"  
  
  - name: Prepare AD
    win_command: D:\Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareAD  /OrganizationName:"Test"
    when: ExchObj_config<16213
    vars:
      ansible_become: yes
      ansible_become_method: runas
      ansible_become_user: TEST\Administrator
      ansible_become_pass: P@$$w0rd
```
We again are using absolutely the same techniques as we did with schema expansion. 
The last step in AD preparation is to prepare domain. 
#### Preparing domain(s)
Domains can be prepared with command.
```cmd
Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareDomain[:<DomainFQDN>]
```
However, if you have only one domain in forest (this is by the way recommended by MSFT), you are already done since /PrepareAD prepares the domain it runs against. 
If it is not the case, you can use the same technique and check objectVersion attribute of Exchange System Objects container. 

Here is how our playbook finally looks like:
```yml
---
- name: Prepare AD
  hosts: ad
  tasks: 
  
  - name: Check .Net version
    win_reg_stat:
        path: HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full\ # required. The full registry key path including the hive to search for.
        name: Release # not required. The registry property name to get information for, the return json will not include the sub_keys and properties entries for the I(key) specified.
    register: netversion

  - name: Install .NET Framework 4.7.1
    win_package:
      path: C:\xfer\NDP471-KB4033342-x86-x64-AllOS-ENU.exe
      product_id: '{0A0CADCF-78DA-33C4-A350-CD51849B9702}'
      arguments: /q /norestart
      state: present
    when: netversion.value < 461310
    register: dotnet_installation
  
  - name: Reboot
    win_reboot:
      post_reboot_delay: 180
    when: (dotnet_installation.reboot_required | default(false))
 
  - name: Install Visual C++ Redistributable Packages for Visual Studio 2013
    win_package:
      arguments: /q /norestart
      product_id: '{929FBD26-9020-399B-9A7A-751D61F0B942}' # not required. The product id of the installed packaged.,This is used for checking whether the product is already installed and getting the uninstall information if C(state=absent).,You can find product ids for installed programs in the Windows registry editor either at C(HKLM:Software\Microsoft\Windows\CurrentVersion\Uninstall) or for 32 bit programs at C(HKLM:Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall).,This SHOULD be set when the package is not an MSI, or the path is a url or a network share and credential delegation is not being used. The C(creates_*) options can be used instead but is not recommended.
      state: present # not required. Whether to install or uninstall the package.,The module uses C(product_id) and whether it exists at the registry path to see whether it needs to install or uninstall the package.
      path: C:\xfer\vcredist_x64.exe 

  - name: Get Schema Version
    win_command:  dsquery * CN=ms-Exch-Schema-Version-Pt,CN=Schema,CN=Configuration,DC=test,DC=local -attr rangeUpper -scope base
    register: rangeUpper
    changed_when: false
    failed_when: "'Directory object not found' not in rangeUpper.stderr and rangeUpper.stderr!=''"
    check_mode: no

  - set_fact:
      Schema: "{{ rangeUpper.stdout_lines[1] | int }}"

  - name: Extend Schema
    win_command: D:\Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareSchema
    when: Schema!='15332'
    vars:
      ansible_become: yes
      ansible_become_method: runas
      ansible_become_user: TEST\Administrator
      ansible_become_pass: P@$$w0rd
    notify:
    - Pausing for replication to occur

  - meta: flush_handlers

  - name: Get Exchange System Object Version
    win_command:  dsquery * "CN=Test,CN=Microsoft Exchange,CN=Services,CN=Configuration,DC=test,DC=local" -attr objectVersion -scope base
    register: Object_version_configuration
    changed_when: false
    failed_when: "'Directory object not found' not in Object_version_configuration.stderr and Object_version_configuration.stderr!=''"
    check_mode: no

  - set_fact:
      ExchObj_config: "{{ Object_version_configuration.stdout_lines[1] | int }}"  
  
  - name: Prepare AD
    win_command: D:\Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareAD  /OrganizationName:"Test"
    when: ExchObj_config<16213
    vars:
      ansible_become: yes
      ansible_become_method: runas
      ansible_become_user: TEST\Administrator
      ansible_become_pass: P@$$w0rd

  handlers:
  - name: Pausing for replication to occur
    pause:
      minutes: 2 # not required. A positive number of minutes to pause for.
```

Let's start the playbook first time:


As we see .Net was installed, machine was rebooted after that, Schema was extended, playbook was paused and then AD was prepared.

And the second one:
![Secondrun.PNG]({{site.baseurl}}/_posts/Secondrun.PNG)

As we see nothing is changed when we are running it the second time, so idemptency test is passed.
