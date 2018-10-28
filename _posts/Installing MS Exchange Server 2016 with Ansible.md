---
published: false
---
## Installing MS Exchange Server 2016 with Ansible

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.

This post is the first one in series of post devoteed to Ansible + Windows. To say more precisely, not just Windows but Ansible + enterpise Apps like MS Exchange Server, SQL Server, etc. 

Ansible is shipped with modules for confiruing different core parts of Windows (files, registry, etc.), some Windows features like IIS but there is no say modules for managing MS Exchange or MS SQL (though there is one module for creation SQL databases). 

This post does not pretend to be best practice on how you would configure MS Exchange or other app, moreover for the sake of simplicity I will not sometimes follow best practices (like creating roles instead of keeping all the tasks in playbook), however it will demonstrate some techniques that can be used in the real life.

Thats being said, let's get started. 
## Environment and Ansible Inventory

## MS Exchange installation process
In order to create playbook we apparently need to understand what steps we need to perform to install MS Exchange Server 2016. 

These steps as well as system requirements are outlined in the following  Technet [section](https://docs.microsoft.com/en-us/Exchange/plan-and-deploy/plan-and-deploy?view=exchserver-2016) and generally include the following steps:
- Installation of prerequisites:
	- On Domain Controller if we want to use domain controller to prepare AD.If not we should onlu care about 		Exchange
    - On Exchange Server itself
- AD changes:
  - Schema expansion 
  - AD preparation 
  - Individual domains preparation
- Installation of Exchange Server


 

