#Linux Active Directory Integration

This role automates the integration of a standalone Ubuntu server into any Active Directory. It adds passthrough authentication to SSH, based on Active Directory user groups, and creates Linux user accounts for all users. The role uses Samba, Winbind, and PAM to join and authenticate users against Kerberos

##Requirements

Ansible dynamic inventory plugins are required as described in https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html. Specifically, the test playbook in this role depends on AWS EC2 dynamic inventory. This playbook looks for its host based on AWS tags.

##Role Defaults and Variables

These required variables are currently in use by the role:

###Global Variables

These required variables describe the Active Directory where the host will become a member.

hostname: Intended hostname of the target server.
    Default: "bastion"
admin_user: A user who has the rights to join / unjoin computers to the Active Directory
    Default: "join"
admin_pasword: The password for the admin user.
    Default: Blank. Suggested use: handle with a vars_prompt (see example playbook).
netbios: NetBIOS name inside the Windows domain
    Default: "EHE"
rfc_version: LDAP RFC version (rfc2307 or rfc2303bis)
    Default: "rfc2307"
ldap_domain: Domain suffix of the Active Directory LDAP server.
    Default: "ehe.exechealthgroup.com"
access_groups: List of groups that should be allowed to log into the server. Multiple groups supported, 1 is required.
    Defaults:
      - vpn-prd
      - Coda Dev Admins

sudo_group: One group that should be allowed sudo access to the server.

###LDAP Server(s)

Hostname(s) of one or more Active Directory LDAP servers. The LDAP server supports Active Directory lookup services. Multiple entries supported. 1 entry is required.

ldap_server:
- host: "hostname"            # Default: "mgmtdc01"
  server_ip: "ip address"     # Default: "10.100.0.200"
  domain: "domain name"       # Default: "{{ ldap_domain }}"

###PDC Server(s)

Hostname of the Active Directory Primary Domain Controller(s). Multiple entries supported. 1 entry is required, multiple supported in a dictionary list. Each server must be accessible and able to join a host to the Active Directory.

  - host: "hostname1"          # Default: "mgmtdc01"
    domain: "ip address2"      # Default: "{{ ldap_domain }}"
    server_ip: "12.34.5.67"    # Default: "10.100.0.200"
  - host: "hostname2"          # Default: "mgmtdc02"
    domain: "ip address2"      # Default: "{{ ldap_domain }}"
    server_ip: "12.34.5.68"    # Default: "10.100.10.200"

###KERBEROS SETTINGS

Information required to authenticate with Kerberos.

kerberos_realm: Realm (domain suffix) of the Active Directory Kerberos authentication server
    Default: "{{ ldap_domain }}"

Kerberos ticket servers in a variable list. Multiple entries supported. 1 entry is required.

kerberos_server:
  - host: "hostname"          # Default: "{{ pdc.0.host }}"
    domain: "ip address"      # Default: "{{ ldap_domain }}"
    server_ip: "domain name"  # Default: "{{ pdc.0.server_ip }}"

###NETWORKING VARIABLES
This role uses Netplan to add layers of configuration onto an existing Ubuntu network stack. For the most part, these may be left at their defaults. But they are exposed for configuration by advanced users.

netplan_interface: Ethernet adapter to be configured
    Default: Primary ethernet adapter
netplan_configured: Enable netplan? ( true | false ) Default: true
netplan_ip: Adapter IP addresses
    Default: Primary private IP address (auto-assigned)
netplan_mac: Adapter MAC address
    Default: Primary adapter's MAC address
netplan_ec2: AWS internal lookup. Defaults are:
  - domain: "ec2.internal"
    server_ip: "127.0.0.53"

###Optional Variables

####unjoin

__Optional:__ a boolean variable for unjoining the active directory. Set in vars/main.yml or as --extra-vars "unjoin=true"

Example: ```AWS_PROFILE=ehe ansible-playbook test.yml --extra-vars "unjoin=true"```

Use when: Replacing the server with a new one.

####kops_deploy

__Optional:__ a boolean variable for adding kubectl and helm clients to the server. Set in vars/main.yml or as --extra-vars "kops_deploy=true"

Example: ```AWS_PROFILE=ehe ansible-playbook test.yml --extra-vars "kops_deploy=true"```

Use when: This server will access and/or manage Kubernetes clusters.

##Dependencies

Currently, this role will only work with Ubuntu.

##Example Playbook
```
- hosts: "192.168.0.1"
  become: True
  become_user: root
  gather_facts: yes
  vars:
    ansible_user: ubuntu

  vars_prompt:
    - name: admin_password
      prompt: "Please enter the password for {{ admin_user }}@{{ ldap_domain|upper }}"
      private: yes
  pre_tasks:
    - name: Install min python if needed
      raw: test -e /usr/bin/python || (apt -y update && apt install -y python-minimal)
      changed_when: false
    - name: "Display all variables/facts known for {{ hostname }}"
      debug:
        var: hostvars[inventory_hostname]
        verbosity: 0
  roles:
    - { role: "linux_ad_integration" }
```

**Example Run Command**
```
AWS_PROFILE=ehe-dev AWS_REGION=us-east-1 ansible-playbook test.yml --extra-vars "kops_deploy=true"
```

##License

Apache 2.0 Version 2.0, January 2004  
http://www.apache.org/licenses/
