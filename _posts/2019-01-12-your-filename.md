---
published: false
---
## How to create your own Ansible module for Windows

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.

In this post we will create very basic Windows module. It will repeat official documentation https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general_windows.html in some areas but it will cover a bit more than documentation covers. 

Ansible module for management some piece of Windows or related application should be written in PowerShell. Since PowerShell is present on all modern Windows OS, nothing needs to be installed on managed node and Ansible will take care about copying all required code including new module's code to the managed node. 

While the module should be written using PowerShell, the module should follow some rule to integrate with Ansible. 
First of all, Ansible uses JSON to communicate with module so:
* do not use _throw_ or _Write-Error_ -  use _Fail-Json_ instead
* do not output anything to host,  and use _Exit-Json_ at the end of your module. 
And always follow PowerShell scripting best practices!

One may ask - where the mentioned _*-Json_ functions originates from?

They come from module https://github.com/ansible/ansible/blob/devel/lib/ansible/module_utils/powershell/Ansible.ModuleUtils.Legacy.psm1 so you will need to indicate that your module requires this one to use its functions:
```PowerShell
#Requires -Module Ansible.ModuleUtils.Legacy
```
This will ensure that Ansible will copy the module to managed node. Besides already mentioned Fail-Json and Exit-Json the module contains some helper functions for parameters processing, objects handling. We will talk about them during implementation of our module. 

## Parameters
Let's assume we want to create module which will create file on file system. 
We need to know the path of the file and the path should be provided by Ansible as parameter for our module.  
Ansible will send JSON object with parameters to our module and in order to consume it and work with it in more convinient way we will use _Parse-Args_ function from the Ansible.ModuleUtils.Legacy module.
```PowerShell
$params = Parse-Args -arguments $args -supports_check_mode $true
```
### Check mode
Ansible playbok can be run in check mode (-C argument). In this mode Ansible will not make changes but will report the possible changes. This alllow you to examine possible changes without applying it. Not all modules support this mode (e.g. win_command, win_shell) but new modules should support it. 

Ansible will let us know if it started in this mode and we can learn this by gettting appropriate parameter
```PowerShell
$check_mode = Get-AnsibleParam -obj $params -name "_ansible_check_mode" -type "bool" -default $false
```

### Diff mode
In diff mode Ansible report not only overall state of the task (i.e. ok or changed) but also shows actual change (i.e. diff in file content, object deletion or creation, etc.)

We can learn if Ansible is running in diff mode 
```PowerShell
$diff_mode = Get-AnsibleParam -obj $params -name "_ansible_diff" -type "bool" -default $false
```