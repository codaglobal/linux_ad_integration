linux_ad_integration
=========

This role automates the integration of a standalone Ubuntu server into any Active Directory. It also adds passthrough authentication to SSH, based on Active Directory user groups.

Requirements
------------

Ansible dynamic inventory plugins are required as described in https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html. Specifically, the test playbook in this role depends on AWS EC2 dynamic inventory. This playbook looks for its host based on AWS tags.

Role Variables
--------------

These required variables are currently in use by the role:

GLOBAL:

hostname: Intended hostname of the target server.
    Default: "bastion"
ldap_host: Hostname of the Active Directory LDAP server.
    Default: "caldc03"
ldap_domain: Domain suffix of the Active Directory LDAP server.
    Default: "ehe.exechealthgroup.com"
ldap_server_ip: IP address of the Active Direcectory LDAP server.
    Default: "192.168.0.200"
pdc1_host: Hostname of the Active Directory primary domain controller.
    Default: "ehenydc01"
pdc1_domain: Domain name suffix of the Active Directory primary domain controller.
    Default: "ehe.exechealthgroup.com"
pdc1_server_ip: IP address of the Active Directory primary domain controller.
    Default: "192.168.0.206"
pdc2_host: Hostname of the Active Directory alternate domain controller.
    Default: "mgmtdc01"
pdc2_domain: Domain name suffix of the Active Directory primary domain controller.
    Default: "ehe.exechealthgroup.com"
pdc2_server_ip: IP address of the Active Directory primary domain controller.
    Default: "192.168.0.206"
netbios: NetBIOS name inside the Windows domain
    Default: "EHE"
kerberos_realm: Realm (domain suffix) of the Active Directory Kerberos authentication server
    Default: Same as pdc1_domain
kerberos1_fqdn: Hostname of the Active Directory Kerberos authentication server
    Default: Same as pdc1_host
kerberos1_server_ip: IP address of the Active Directory Kerberos authentication server
    Default: Same as pdc1_server_ip
kerberos2_fqdn: Hostname of the Active Directory Kerberos authentication server
    Default: Same as pdc1_host
kerberos2_server_ip: IP address of the Active Directory Kerberos authentication server
    Default: Same as pdc1_server_ip
rfc_version: LDAP RFC version (rfc2307 or rfc2303bis)
    Default: "rfc2307"

PACKAGES:

newpackages: Packages to be installed on launch. Add or subtract from this YAML list as appropriate.
Defaults:
  - ntp
  - ntpdate
  - winbind
  - samba
  - libnss-winbind
  - libpam-winbind
  - krb5-config
  - krb5-locales
  - krb5-user

NETWORKING VARIABLES:
This role uses Netplan to add layers of configuration onto an existing Ubuntu network stack. For the most part, these may be left at their defaults. But they are exposed for configuration by advanced users.

netplan_interface: Ethernet adapter to be configured
    Default: Primary ethernet adapter
netplan_configured: Enable netplan? ( true | false ) Default: true
netplan_ip: Adapter IP addresses
    Default: Primary private IP address (auto-assigned)
netplan_mac: Adapter MAC address
    Default: Primary adapter's MAC address
netplan_ns1: Nameserver for DNS lookup.
    Default: LDAP server's IP address
netplan_ns2: Nameserver2 for DNS lookup.
    Default: Primary Domain Controller's IP address
netplan_ns3: Nameserver3 for DNS lookup.
    Default: Secondary Domain Controller's IP address.
netplan_ns3: Nameserver4 for DNS lookup.
    Default: Amazon EC2 internal.
netplan_dns1: DNS search domain
    Default: LDAP DNS suffix
netplan_dns2: DNS search domain 2
    Default: EC2 internal

UNJOIN VARIABLE:

Optional: a boolean variable for unjoining the active directory. Set in vars/main.yml or as --extra-vars "unjoin=true"

Example: ansible-playbook test.yml --extra-vars "unjoin=true"

Use when: Replacing the server with a new one.

Dependencies
------------

Currently, this role will only work with Ubuntu.

Example Playbook
----------------

- hosts: "tag_Name_ehe_management_{{ hostname }}"
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

License
-------

Apache 2.0
