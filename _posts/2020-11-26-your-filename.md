---
published: false
---
## Installing MS Exchange Server with Ansible. Part 1: Preparing Active Directory

This is first part of the series devoted to installation and management of MS Exchange Servers with Ansible. 
Why Ansible?
Ansible is very powerfull automation tool. It is FOSS, agentless and is able to manage Windows, Linux, network devices, etc. 
As for Windows it uses either winrm or psrp on top of it. It can also use SSH for Windows too but it is experimnental. 
Ansible cames with bunch of modules for Windows management. Of course, Ansible does not have modules for every system or scenario you want to automate. 
But it can use DSC modules and it is easy to create your own Ansible module. 
The latter fact is very attractive for Windows administrator as module for Windows is just PowerShell script  and Ansible provides libraries for easy integration.
Ansible module is way simpler than say DSC module which requirements are different for PowerShell 4.0 or PowerShell 5.1. The last one requires you to create classes that implement DSC methods. While OOP is nice it may look overcomplicated for administrator. 
Ansible module does not have many requirements, it's inteface is simple: JSON in - JSON out. Also, Ansible would work with any PowerShell 3.0 version so you can use to manage old systems that for some reasons can't be upgraded to newer PowerShell. 
So would it be article on how to use Ansible to invoke DSC modules for Exchange? 
While it is posible, the answer is no. We  will use our own modules. 
Why? For many reason. 
* it is interesting to create them :)
* you may want to have functionality DSC does not provide
* you are interested in have control over the code doing the changes

### Starting point and disclaimers

We will asume that we have the following in place:
* AD is up and running
* it consists of one forest with one domain
* server for Exchange installation is just fresh installed Windows joined to the domain

_Disclaimer:_ what is described here is just POC and reasearching project. 
It may not be in compliance with best practices from security and performance perspective.


### Base Ingridients
 
Ansible requires 3 components:
* inventory  (host, groups list and vars)
* role(s) (container with tasks)
* playbooks (basically link between the 2 above)

#### Inventory

Our inventory would be quite simple:
* 1 domain controller
* 1 Exchange server

It will also contain connection variables, service accounts we will use for management, etc. 

###  Playbook

### Role

We will use the role from this repo for configuring Exchange. It proved to be working with Exchange 2016 and Exchange 2019 but in case you want to use, you should definitely test it in your environment. 
We will look in the role's taks and modules in greater details. 

## Flow

So what we need to do to prepare AD?

We need to: 
* Expand Schema
* Prepare AD
* Prepare Domain

These steps can be run from domain controller or from our future Exchange server.
All these steps imply running setup.exe utility from Exchange installation media. 
The utility has some requirements to run. So we have the following action items:
* provide Exchange installation media
* install prerequisites for the tool

### ISO

ISO with evaluation version can be downloaded from Microsoft evaluation site.
As we work with virtual machines, we can assume that we have attached ISO to VM and it is accessivble in the system. Basically, Ansible can talk to hypervisors too to implement this step, but it is beyond the scope of the article. 
Also media can be copied to local machine's disk. 
The path to the setup.exe is set in our role's defaul variables. 

### Prerequisites

Per [Exchange Server prerequisites](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/prerequisites?view=exchserver-2016#exchange-2016-prerequisites-for-preparing-active-directory) we need to install the folowing components on our server:
- .NET Framework 4.8
- Visual C++ Redistributable Packages for Visual Studio 2013
- RSAT-ADDS

We can install the first 2 components with win_package module but we will to take care about delivering software distributives with Chocolatey. 
The last component can be installed with win_feature module.
So this part would look like:

```yml
```

### Schema Expansion

Schema expansion is basically done by running:
```cmd
Setup.exe /IAcceptExchangeServerLicenseTerms /PrepareSchema
```
The command should be run by member of Schema Admin and Enterpsie Admin security groups.

This command behavior does not suit very vell for idempotancy purposes - even if the schema is already expanded, the second run of this command will still report that schema is successfully expanded. Thus we need to find the way to determine if the schema is already in a good shape. 
The answer is [here](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/active-directory/ad-changes?view=exchserver-2016#extend-the-active-directory-schema):
> After the schema has been extended by running the /PrepareSchema command, the _/PrepareAD command, or installing the first Exchange server using the Exchange Setup wizard, the schema version is set in the ms-Exch-Schema-Version-Pt attribute. To verify that the Active Directory schema was extended successfully, you can check the value stored in this attribute

In addition, this [article](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/prepare-ad-and-domains?view=exchserver-2016#exchange-active-directory-versions) contains the values of this attribute. If we are installing CU10, it will be 15532. In order to check the attribute value, we could use Ansible's win_command module with good old _dsquery_.  

```yml
  - name: Get Schema Version
    win_command:  dsquery * CN=ms-Exch-Schema-Version-Pt,CN=Schema,CN=Configuration,DC=test,DC=local -attr rangeUpper
    register: rangeUpper
    changed_when: false
    failed_when: "'Directory object not found' not in Schema_version.stderr and Schema_version.stderr!=''"
    check_mode: no
```

The better option is to create module that would collect this data for us and report it the way we could use it in Ansible without further modifications. 

So we created very simple module, its code can be inspected in and it returns 2 values indicating if the schema is expanded and the value of mentioned attribute. 

How would we use this data?
We can have the value of the attribute specified in variables and invoke the Setup.exe only when it is required. 

As we already noted above, we need to run the this command in the security context of account with Schema Admin permissions. 
But if we just use this account as our ansible_user, we will hit famous double-hop issue of remoting. 
There are several ways to resolve it:
* use _become_. In this case we use ansible_user to connect to the node via winrm but then Ansible will start another PowerShell process in security context of ansible_become_user
* use authentication protocol that allows to delegate credentials to managed machine. In this case ansible_user fresh credentials will be avaialable to the remote host:
** credssp
** kerberos delegation (technically, there will not be credentials but forwardable kerberos ticket).

There are many warnings about credssp usage due to the fact of passing credentials to remote hosts but basically any of this methods will pass credentials to remote host in different forms, and if you have security concerns well you should not do it in any way. 
Alternatively, you can talk directly to schema master to avoid double hop issue and expand schema directly on it.

In this case we will use become with appropriate account. 
The account is specified in role's default but you must definitely override it on playbook or inventory level dynamically pulling the sensitive data from secrets storage system like HashiCorp's Vault or encrypt it with Ansible's vault and provide vault key when required. 

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


Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
