
# ZDI Overview
This readme provides details regarding Win10 ZDI/VDI automation with Ansible and Ansible Tower architecture design, configuration, administration tasks and contextual descriptions of the project. The following are the main objectives of the project:
* Self-service ZDI provisioning with lifecycle management
...VDI provisioning
...utomatically VDI deletion
* ZDI/VDI lease extension – Self-service Snooze button
* Self-service VDI deletion
* ZDI/VDI status reporting

Win10 ZDI/VDI automation with Ansible/Ansible Tower has the following high level requirements:

* Entitled business users will request a VDI desktop from Ansible Tower as the self-service portal.
* Given that this is a pre-approved single VDI desktop use case Image, the VDI desktop will be auto-provisioned and an email notification will be sent to the requesting business user that it is ready for use.
* After 30 days, the requesting business user will be sent an email notification and asked if they would like to extend the use of their VDI desktop for another 30 days. 
...If the business user takes appropriate action to acknowledge the extended use, an automated schedule job will extend the VDI desktop. An email notification will be sent to the business user of their new termination date.
...If the business user does not take appropriate action to acknowledge the extended use, the VDI desktop will be automatically de-provisioned. An email notification will be sent to the business user that their VDI desktop has been decommissioned.

## High Level Architecture

In the current phase of the ZDI/VDI automation project, the front end is the Ansible Tower, the business user will login to the Ansible Tower to initiate a tower job to create Win10 VDI desktop. The Ansible Tower Create Win10 VDI job template backend Ansible playbook will use playbook library to start the process, eventually the playbook will communicate to a VMware vCenter, and create a VM based on specified VM template and customization spec, so a VDI desktop is created, and an email notification will send to the business user who requested the VDI. 
At the meantime, utilizing the same infrastructure described above, there is a Win10 VDI Reclamation job scheduled in Ansible Tower, running once every day. This job will fulfill VDI lifecycle management task, which is deleting VDI after 30 days of its creation. 
There are two other self-service service catalog tasks that allow business user to extend the lease of VDI desktop or delete VDI desktop before reclamation.  

In a future phase, new self-service portal utilizing ServiceNow can be plugged into the current Ansible/Ansible Tower implementation. The change will only be in the front end portal portion, the backend as well as the current front end will be kept intact shown from below graphic.


# VDI Automation Ansible Playbooks and Roles

## Ansible Code Structure in VDI Automation

VDI automation Ansible code consists two main folders, one is playbooks, and another one is roles. This is high level folder structure:

### Playbooks
Playbooks folder includes high level playbooks which will be used by Ansible Tower job templates. They are 5 playbooks, corresponding to the 5 job templates in Ansible Towers for VDI automation:

```
09/06/2021  03:15 AM               768 vdi_decom.yml
09/06/2021  03:15 AM               938 vdi_extend.yml
08/22/2021  03:08 AM             2,451vdi_provision.yml
08/31/2021  02:59 PM             1,369 vdi_report.yml
08/13/2021  02:00 PM               773  vdi_schedule.yml
```

* vdi_provision: provisioning a ZDI and update ZDI metadata under annotation to determine when to decommission the ZDI after 30 days.
* vdi_schedule: scan all live ZDIs to determine the playbook will send an email notification on expiration of a VDI, or decommssion the VDI.
* vdi_extend: extend the expiration day of specified ZDI(s).
* vdi_decom: delete the specified ZDI(s).
* vdi_report: send ZDI lease status report by email, scan all live VDIs to list lease expiration date and remaining lease hours, etc.


### Ansible Roles

Role is a set of tasks and additional files to perform certain task. Roles are ways of automatically loading certain vars_files, tasks, and others based on a known file structure. Grouping content by roles also allows easy sharing of roles with different playbooks. 

Roles expect files to be in certain directory names. Roles must include at least one of these directories, however it is perfectly fine to exclude any which are not being used. When in use, each directory must contain a main.yml file, which contains the relevant content.

VDI automation has 4 roles currently:

```
08/02/2021  04:20 AM    <DIR>          tower_login_user
07/27/2021  04:13 AM    <DIR>          tower_restapi
08/03/2021  05:25 AM    <DIR>          vdi_name
08/03/2021  05:25 AM    <DIR>          vdi_ops
```

Vdi_name role is used to generate VDI name, vdi_ops role is used for all actions required for VDI automation. Tower_login_user role is simply used to get Ansible Tower login user name and job ID. Currently we are not using tower_restapi role in VDI automation.  

As you can see all of them has “defaults” and “tasks” folder. The “defaults” folder has main.yml file which defines default variables used by that role.

### Ansible Variables in VDI Automation

Variable can be defined from 2 places for VDI automation, one is in Ansible Tower Job Template Extra Variables field, another one is defined in Ansible playbook, in case of VDI automation, we should define them in main.yml file under “defaults” folder of roles.
 
As mentioned above, the “defaults” folder of each Ansible role has main.yml file which defines default values of variables used in Ansible role and Ansible playbook. The default variables in vdi_ops role has majority of the variable definition for VDI automation:

To change the variable value, either modify the default value in the Ansible code, or change the extra variables in the Ansible Tower job template as described earlier.

### VDI_OPS Role in VDI Automation
In addition to defaults and tasks folder, vdi_ops role also has library and template folders.

* calc_lease.yml	
...Tasks for calculating the lease period and expiration date - notify_warning, pouplate_info
* decom.yml	
...Decommission VDI using Ansible Vmware module, as well as sending email notification - ondemand_decom, reclamation
* email.yml	
...Send email - decom, notify_warning, notify_provision, notify_report
* enum_vdi.yml	
...enumerate all VDIs under specified folder - live_scan (in vdi_name), vdi_report, vdi_schedule

* extend_lease.yml	
...Extend a lease of single VDI - vdi_extend
* find_vdi.yml	
...Find a VDI by name by querying vCenter - vdi_provision, extend_lease, ondemand_decom
* get_vdi_info.yml	
...Get VDI information from VM's annotation field - extend_lease
notify_provision.yml	
...Generate email content, and send email after VDI provisioning - vdi_provision
notify_report.yml	
...Generate HTML email content, and send lease status report by email - vdi_report
* notify_warning.yml	
...Generate warning email content an send lease expiration warning email - reclamation
* ondemand_decom.yml	
...Decommission a single VDI manually - vdi_decom
* populate_info.yml	
...Populate single VDI information to a list from VM's annotation - vdi_report
* provision.yml	
...Provision or modify a VDI by using Ansible Vmware module - vdi_provision
* reclamation.yml	
...Tasks for scanning a single VDI to determine if it needs to be deleted, or send email notification- vdi_schedule

#### Library:
Vdi_ops library includes two Pathon files that overwrite the two default Ansible modules, those become Custom Ansible Modules designed for VDI automation. Two modules are vmware_guest, and vmware_vm_facts.

Here is the list of reasons why we need to have custom Ansible Modules
* vmware_guest: VDI provisioning:
...Unable to specify datastore without disk size, etc. 
...Unable to specify customization spec (upcoming Ansible 2.6 will have that issue fixed)       
* vmware_vm_facts: Enumerate VDIs
...Unable to specify a folder
...Slow to run
...Unable to retrieve VM annotation

It is safe to contine using those two custom Ansible modules in forseeable future even if the Ansible version is updated to newer version in tower. Currently Ansible Tower version in PROD is 2.4.3, the FTDEV Ansible Tower version is 2.5.2.

The custom Ansible module path is currently defined under ansible.cfg file: 

```
[defaults]
roles_path = ./roles
library = ./roles/vdi_ops/library 
```


If new custom Ansible module needs to be added to the VDI automation project, it should be added under /vdi_ops/library, insetad of default /playbooks/library. 

#### Templates:
Vdi_ops template folder includes all email notification templates and VM annotation JSON template.


# VDI Automation Design and Operation Review

## Ansible Tower Job Templates Overview
As stated earlier, VDI automation objectives are:
* Self-service VDI provisioning with lifecycle management
...VDI provisioning
...Automatically VDI deletion
* VDI lease extension – Self-service Snooze button
* Self-service VDI deletion
* VDI status reporting

Based on those requirements, Ansible Tower job templates for VDI automation provide 4 self-service job templates and 1 scheduled job templates. 

An Ansible job template provides simple user interface to launch a job, which is basically to run an Ansible playbook. An Ansible playbook is a series of tasks, for example, creating a VDI desktop, save VDI metadata information to VM’s annotation field, and send email notification to the end user who requests the VDI, etc.

## Create a Win10 VDI Desktop Job Templates 

Basically this job template provisions a Win10 VDI desktop, then send an email notification about the newly provisioned VDI desktop.

### User Interface
There is one mandatory field, which is the email address of the VDI owner. This is to determine who the owner of the VDI is. You can provide other email addresses that will receive email notification about activities of this VDI, including lease expiration, lease extension, and deletion. VDI note/label is a good way to identify and track a VDI in the future email notifications through the lifecycle of the VDI. All email notifications later on with regards to this VDI will show the original VDI note/label. For example, “Phase 2 test VDI for new Win10 hello-world app”. 

The design enables requesting VDI on behalf of somebody else by providing owner email address. Ansible Tower login user is not the owner of the VDI. The Ansible Tower login user requesting the VDI will not get email notification if he/she is not the owner, or his/her email address is not manually added to the email cc list in the request form/survey. However, Ansible Tower login user name and Ansible Tower job ID will be logged in the email notification, so the owner will know who requested the VDI, and the job ID can be used to track the backend activities related to this request.

## Win10 VDI Reclamation Job Templates 

Basically this job template scans all live Win10 VDI desktop, then send email notification for those VDIs that are about to expire, or delete expired VDIs.

### User Interface

There is no survey (user interface) for this job template. The template can be run manually at any time, but it is scheduled to run automatically once every day at 2 AM Los Angeles/Pacific Time. 

If this schedule time needs to be changed, the job template needs to add extra variables in the following format:
schedule_run_time: 02:00:00 

### Email Notificaiton

There are total of 3 warning emails sent to the original owner of VDIs and original VDI email cc list specified during the VDI provisioning, when the VDI is: 
* 3 days before lease expires
* 2 days before lease expires, same content as above
* 1 day before lease expires, same content as above, but with different subject line: “last reminder” 

## Extend Win10 VDI Lease Job Templates 

This job template extends the lease of one or more Win10 VDI desktop, then sends email notification about its new expiration date.

### User Interface

The survey allows multiple VDIs to be extended all at the same time. The user needs to specify the name of VDI(s) to be extended. 
In summary:
* The business user should run this when he/she gets warning email notification about lease expiration.
* Specify VDI name(s) to extend the lease
...Can extend a VDI on behalf of somebody else
* Specify lease period between 3 to 15 days
* You’re allowed to shrink a lease when the original lease period is still longer than the new lease period
* Email notification will send to the original owner and original VDI email cc list. The requestor – Ansible Tower login user - will not get email notification if his/her email address is not in the original email cc list, and he/she is not the owner of the VDI.
* Ansible Tower login user name and Ansible Tower job ID will be logged in the email notification, so the owner will know who requested the VDI, and the job ID can be used to track the backend activities related to this request. 
* There is no limit on how many times the business owner can extend the lease of a VDI. 

You may see error in the report when the operation will shrink the lease of VDI, but the error is ignored.

## Delete Win10 VDI Desktop Job Templates 

This job template allows Win10 VDI desktop(s) to be deleted manually without depending on VDI reclamation schedule job.  

### User Interface

* Specify VDI name(s) to delete 
...VDI name will be validated
...Can delete a VDI on behalf of somebody else
* Delete will happen immediately without any cooling-off period
* Email notification will send to the original owner and original VDI email cc list
* The requestor initials will be logged in the email notification, as well as Ansible Tower Job ID.


## Win10 VDI Desktop Status Report Job Templates 

This job template provides a list of all Win10 VDI desktops inventory in the system, and VDI lease status, that allows administrator to get real time information about leases of all VDIs.

### User Interface 

A few notes about the Survey:
* All 3 fields are mandatory
* Email address is the where the report will be sent to. Multiple emails can be entered separated by semicolon.
* Sort By is used to sort column of the status report table, so it can be better viewed on interested field. The options are:
...VDI name (default)
...Remaining hours before VDI lease expires
...Lease period of the VDI
...Owner name
* List all columns (true/false) is used to determine if all VDI metadata will be shown in the report or not. By default it is false. If true, the following columns will be included in the report:
...Owner email address
...VDI note
...Email cc list

Additional note about this feature:
* It can be run at any time, the real time information will be gathered, such as remaining hours of lease, and expiration date
* A job can be scheduled to run this job template once a day to send the report to administrator automatically from Ansible Tower.
* To schedule a job, make sure the following extra variables are passing into the schedule. 










