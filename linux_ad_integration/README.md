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

The following boolean variables are set in defaults/main.yml, overridden by vars/main.yml or via the --extra-vars option on the Ansible command line.

####unjoin

__Optional:__ Use this variable to unjoin a server from the Active Directory.

**Default:** false

Example: ```AWS_PROFILE=ehe ansible-playbook test.yml --extra-vars "unjoin=true"```

Use when: Replacing or taking down the server.

####ansible_install

__Optional:__ Use to install Ansible on the target server with preferred settings

**Default:** false

Example: ```AWS_PROFILE=ehe ansible-playbook test.yml --extra-vars "ansible_install=true"```

Use when: Needed to change global roles path to /etc/ansible/roles

Modify in tasks/ansible-install.yml

####ssh_reset

__Optional:__ When joining the Active Directory, SSH is restarted, and this usually interferes with remote Ansible connections. Use the ssh_reset logic to ensure that SSH restarts gracefully on the Ansible source, and that the session to the Ansible target is not lost.

**Default:** true

Example: ```AWS_PROFILE=ehe ansible-playbook test.yml --extra-vars "unjoin=true"```

Use when: Running this role on a remote server.
Set to false when: Running this role locally, especially while connected to a server via SSH.

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
