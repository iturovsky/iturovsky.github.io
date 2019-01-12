---
published: false
---
## How to create your own Ansible module for Windows

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.

In this post we will create very basic Windows module. 

Ansible module for management some piece of Windows or related application should be written in PowerShell. Since PowerShell is present on all modern Windows OS, nothing needs to be installed on managed node and Ansible will take care about copying all required code including new module's code to the managed node. 

While the module should be written using PowerShell, the module should follow some rule to integrate with Ansible. 
First of all, Ansible uses JSON to communicate with module so:
* do not use _throw_ or _Write-Error_ -  use _Fail-Json_ instead
* do not output anythin to host, use _Output-Json_ instead

