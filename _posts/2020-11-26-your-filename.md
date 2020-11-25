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

### Base Ingridients
 
Ansible requires 3 components:
* inventory  (host, groups list and vars)
* role(s) (container with tasks)
* playbooks (basically link between the 2 above)

#### Inventory


Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
