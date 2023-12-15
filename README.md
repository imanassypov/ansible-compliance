# ansible-compliance
 This is a collection of ansible based examples to validate various compliance checks with Cisco IOS devices.
 ansible-compliance.yml: Ansible playbook to validate running configuration compliance against intended state on cisco IOS devices
 ansible-dnac-advisory.yml: Ansible playbook to lookup hostnames of devices managed by Cisco DNA Center which are subject to a specific security advisory

# Intended use-cases
 When running a large set of network devices in your environment it becomes important to maintain configuration consistency and compliance of those devices to the 'intended state' or golden references that have been established within the organization. 
 This ansible playbook demonstrates how compliance validation can be accomplished, and reported on.

 Similarly, when a PSIRT is published it becomes necessary to automate the process of mapping security advisory notices to devices in your network that may be exposed to it. 

## Scope
 This ansible playbook has been tested against Cisco IOS-XE Catalyst 8000v instances deployed on Cisco Modeling Labs (CML) platform.
 Ansible PSIRT lookup playbook is tested against Cisco DNA Center 2.3.5.5 lab with about a dozen of devices under its management

# ansible-dnac-advisory.yml
## pre-requisites
Install required pip modules
```
pip3 install -r requirements.txt
```

Populate the list of PSIRT advisory id's in advisory_list.yml.
Note that the advisory id string can be found in "Advisory ID" field, ie ref: https://sec.cloudapps.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-aaa-Yx47ZT8Q

```
---
advisories:
 - cisco-sa-aaa-Yx47ZT8Q
 - cisco-sa-20180328-dhcpr2
 ```
Create a local file .env and populate with your DNAC address, username and password.
Before executing the script, load required env vars into your shell

```
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
export DNAC_HOST="x.x.x.x"
export DNAC_USERNAME="username"
export DNAC_PASSWORD="password"
```

Load env vars
```
source .env
```

Execute playbook
```
ansible-playbook ansible-dnac-advisory.yml 
```
# ansible-compliance.yml
## Compliance Reference or Intended state
 Let us assume that we want to ensure that every Cisco IOS device in our environment complies with following rules:
 1. Have a single NTP server defined. Assume NTP server IP Address of '10.0.0.252' must be defined on each device.
 2. Have the NTP server secured by an Access Control List, only allowing that device to synchronize its local time with the defined NTP server. Assume NTP ACL named NTP-Server-ACL must be present on each device, and has NTP server IP address '10.0.0.252' allowed.
 3. Have global configuration level best practices applied. In this case, we demonstrate an example of:
    - disable domain lookup (CLI: 'no ip domain lookup' must be present on each device.)
    - disable gratuitous arp (CLI: 'no ip gratuitous-arps' must be present on each device)

## Target device inventory
 For the sake of simplicity we will assume that we have a static text file containing the IP addresses of all of our target IOS devices. In production for an at-scale deployment, you can also utilize sources of dynamic inventory such as Cisco DNA Center, and leverage DNA Center Ansible Inventory plugin to achieve automated access to your device inventory
 
## Example compliant IOS-XE configuration
```vtl
C-C8Kv-CE01#sh run | s ntp|NTP-Server-ACL
ip access-list standard NTP-Server-ACL
 10 permit 10.0.0.252
ntp access-group peer NTP-Server-ACL
ntp server vrf MGMT 10.0.0.252
```
```vtl
C-C8Kv-CE01#sh run | s domain|gratuitous
no ip gratuitous-arps
no ip domain lookup
```
## Example 1. Compliant Ansible execution output

```vtl
ansible-playbook -i hosts ansible-compliance.yml

PLAY [IOSXE Ansible Compliance Validation] ****************************************************************************************************************************************************************************

TASK [Checking NTP ACL] ***********************************************************************************************************************************************************************************************
ok: [10.0.1.61]
ok: [10.0.1.62]

TASK [Checking NTP] ***************************************************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

TASK [Checking best-practice global CLI entries] **********************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

PLAY RECAP ************************************************************************************************************************************************************************************************************
10.0.1.61                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.0.1.62                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```
## Example 2. NTP server is misconfigured
 Let us misconfigure the IP Address of the NTP server on one of the target Cat8k devices, and replace the ip address with '1.1.1.1'. 
 Non-compliant section of the configuration would look like following:

```vtl
C-C8Kv-CE01#sh run | s ntp
ntp access-group peer NTP-Server-ACL
ntp server vrf MGMT 1.1.1.1
```

Sample ansible playbook output detecting non-compliant NTP configuration would look like below in this example

```vtl
ansible-playbook -i hosts ansible-compliance.yml

PLAY [IOSXE Ansible Compliance Validation] ****************************************************************************************************************************************************************************

TASK [Checking NTP ACL] ***********************************************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

TASK [Checking NTP] ***************************************************************************************************************************************************************************************************
ok: [10.0.1.62]
changed: [10.0.1.61]

TASK [Checking best-practice global CLI entries] **********************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

RUNNING HANDLER [NTP compliance violation] ****************************************************************************************************************************************************************************
ok: [10.0.1.61] => {
    "msg": [
        "NTP config compliance violation on 10.0.1.61"
    ]
}

PLAY RECAP ************************************************************************************************************************************************************************************************************
10.0.1.61                  : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.0.1.62                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Example 3. NTP ACL is misconfigured
 Let us misconfigure the permit statement of NTP access list

 ```vtl
C-C8Kv-CE01#sh run | s access-list
ip access-list standard NTP-Server-ACL
 10 permit 1.1.1.1
 ```
Sample ansible playbook output detecting non-compliant NTP Access List configuration would look like below in this example

 ```vtl
 ansible-playbook -i hosts ansible-compliance.yml

PLAY [IOSXE Ansible Compliance Validation] ****************************************************************************************************************************************************************************

TASK [Checking NTP ACL] ***********************************************************************************************************************************************************************************************
ok: [10.0.1.62]
changed: [10.0.1.61]

TASK [Checking NTP] ***************************************************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

TASK [Checking best-practice global CLI entries] **********************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

RUNNING HANDLER [NTP ACL compliance violation] ************************************************************************************************************************************************************************
ok: [10.0.1.61] => {
    "msg": [
        "NTP ACL compliance violation on 10.0.1.61"
    ]
}

PLAY RECAP ************************************************************************************************************************************************************************************************************
10.0.1.61                  : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.0.1.62                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Example 4. Global config best practice CLI missing
 Let us permit gratuitous ARP on one of the devices, note (ip gratuitous-arps CLI)

```vtl
C-C8Kv-CE01#sh run | s domain|gratuitous
ip gratuitous-arps
no ip domain lookup
```

Sample ansible playbook output detecting non-compliant global CLI statements in running configuration,

```vtl
ansible-playbook -i hosts ansible-compliance.yml

PLAY [IOSXE Ansible Compliance Validation] ****************************************************************************************************************************************************************************

TASK [Checking NTP ACL] ***********************************************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

TASK [Checking NTP] ***************************************************************************************************************************************************************************************************
ok: [10.0.1.62]
ok: [10.0.1.61]

TASK [Checking best-practice global CLI entries] **********************************************************************************************************************************************************************
ok: [10.0.1.62]
[WARNING]: To ensure idempotency and correct diff the input configuration lines should be similar to how they appear if present in the running configuration on device
changed: [10.0.1.61]

RUNNING HANDLER [Best-practice CLI compliance violation] **************************************************************************************************************************************************************
ok: [10.0.1.61] => {
    "msg": [
        "Best practice global CLI compliance violation on 10.0.1.61"
    ]
}

PLAY RECAP ************************************************************************************************************************************************************************************************************
10.0.1.61                  : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.0.1.62                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```