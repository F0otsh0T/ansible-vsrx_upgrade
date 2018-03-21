## Overview
This documents in Contrail 3.x & Contrail 2.x environments (via "Roles"), how to perform In-Service upgrade of vSRX Service Instance vFW's (ECMP Scale-Out Active-Active Scenario) in an automated fashion to whichever Juniper vSRX JUNOS version is required. Active-Standby upgrades are currently in development. In our Sandbox DEV lab, it took about ~1.5 hour per vSRX VM upgrade via the Upgrade method and ~20min via the Rebuild method.

* __Active-Active__
```mermaid
graph TD;
  NetworkA-->vFW01-Active;
  NetworkA-->vFW02-Active;
  NetworkA-->vFW03-Active;
  NetworkA-->vFW04-Active;
  vFW01-Active-->NetworkB;
  vFW02-Active-->NetworkB;
  vFW03-Active-->NetworkB;
  vFW04-Active-->NetworkB;
```

* __Active-Standby__ (in development)
```mermaid
graph TD;
  vFW01-Active---vFW02-Standby;
  NetworkA-->vFW01-Active;
  NetworkA-->vFW02-Standby;
  vFW01-Active-->NetworkB;
  vFW02-Standby-->NetworkB;  
```

## Version 1.3
This version of the playbooks is intended to work with Ansible 2.4.x and Juniper.JUNOS stdlib 1.4.3

## Methods
* __REBUILD__: This uses the Ansible 2.4+ Cloud Module os_server_actions and calls the "rebuild" which is analogous to the "nova rebuild" command. This requires a QCOW2 image be preconfigured and available in your TENANT with FXP0 set to dhcp-client and a user created for automated admin access (SSH-KEY or User/Pass) to reconfigure the vSRX VM with a previously saved config.
  - Rebuild method is supported in Contrail 2.x and 3.x environments via the TAGS __'--tags "c2x,rebuild"'__ and __'--tags "c3x,rebuild"'__ input upon running the playbook __c2x-vsrx_si_haaa.yml__.
* __UPGRADE__: This uses the Juniper.junos_install_os Module to push the TGZ upgrade binaries to the vSRX VM and then run install scripts.
  - Upgrade method is supported in Contrail 2.x and 3.x environments via the TAGS __'--tags "c2x,upgrade"'__ and __'--tags "c3x,upgrade"'__ input upon running the playbook __c3x-vsrx_si_haaa.yml__.

## Tags
Instead of using separate __ROLES__ to differentiate between types of vSRX Code Migrations, using __TAGS__ allows us to consolidate the various tasks into a single __ROLE__. Below are some TAGS we have used:

* __c2x__: Contrail 2.x
* __c3x__: Contrail 3.x
* __rebuild__: Migrate vSRX Code via REBUILD Method
* __upgrade__: Migrate vSRX Code via UPGRADE Method
* __reboot.c2x__: Reboot vSRX in Contrail 2.x
* __reboot.c3x__: Reboot vSRX in Contrail 3.x
* __vmi_down.c2x__: Shut down Service-Chained VMI's in Contrail 2.x
* __vmi_down.c3x__: Shut down Service-Chained VMI's in Contrail 3.x
* __vmi_up.c2x__: Admin up Service-Chained VMI's in Contrail 2.x
* __vmi_up.c3x__: Admin up Service-Chained VMI's in Contrail 3.x

## GitHub:
> https://github.com/F0otsh0T/ansible-vsrx_upgrade

## OpenStack Snapshot VM and Convert to QCOW2 Image

> https://docs.openstack.org/mitaka/user-guide/cli_use_snapshots_to_migrate_instances.html

## System and Software Requirements
It is recommended that on your system that your installing Ansible that you correctly have proxy settings set globally on the system.  In the case of Ubuntu you will want to edit the /etc/environment file and include the below strings for proxy to work.  Use of the apt-get and ansible-galaxy will leverage these proxy settings accordingly;

http_proxy=http://proxy.com:8080/
ftp_proxy=ftp://proxy.com:8080/
https_proxy=http://proxy.com:8080/

  >  * sudo -H apt-get install software-properties-common
  >  * sudo -H apt-get install python
  >  * sudo wget https://raw.github.com/pypa/pip/master/contrib/get-pip.py and/or wget https://bootstrap.pypa.io/get-pip.py
  >  * sudo -H python get-pip.py
  >  * sudo -H apt-add-repository ppa:ansible/ansible
  >  * sudo -H apt-get update
  >  * Python >= 2.7 Python Modules (use pip to install)
  >    - sudo -H pip --proxy=proxy.com:8080 install shade --process-dependency-links
  >      + May require updated pbr
  >      + May require lxml
  >      + May require libssl-dev
  >      + May require libffi-dev
  >      + May require updated pip
  >  * sudo -H pip --proxy=proxy.com:8080 install jxmlease --process-dependency-links
  >    - junos-eznc
  >    - junos-netconify
  >    - ncclient
  >    - pycrypt
  >    - cryptography
  >  * sudo -H apt-get install ansible
  >    - Ansible >= 2.4.0 (or use PIP to install Ansible)
  >    - (Optional) ansible-doc
  >    - Ansible Module (use ansible-galaxy): Juniper.junos >= 1.4.3
  >  * sudo -H sudo ansible-galaxy install git+https://github.com/Juniper/ansible-junos-stdlib.git,,Juniper.junos
  >  * sudo -H apt-get install whois (for mkpasswd)

## Prerequisites

*Accounts for Juniper and OpenStack (with Admin access to shut down OS::Neutron::Ports)
  - Ansible OpenStack Modules: OpenStack Keystone account with Admin Access to your Tenant & Neutron Networks / Ports. This will be used to disable and re-enable OS::NEUTRON:PORT(s) during the upgrade.
  - Ansible Juniper_OS Modules: Local vSRX account or AAA account with SSH Keys configured. This will be used to perform operations on the vSRX itself during the upgrade.

* Create SSH Keys for MechID account on Linux host
  - Naming of Public and Private keys are important due to a bug in the Ansible Juniper.junos module ( https://github.com/Juniper/ansible-junos-stdlib/issues/85 ). The bug the prevents usage of keys named anything other than:
    ```ssh
    Private: id_rsa
    Public: id_rsa.pub
    ```
  - Please note: If you use anything other than the stock/default id_rsa* keys in your ~/<home>/.ssh directory, you will need to specify the location and files of the SSH keys in the ansible.cfg file or via command line during ansible-playbook execution.

* Configure vSRX with MechID account with SSH-KEY and NetConf via SSH (Alternatively, use User/Pass instead of SSH-KEY).
  - vSRX user should have full access / admin rights to perform upgrades.
  - E.g.
    ```junos
    system {
          login {
              user username {
              full-name "Ansible and Automation MechID";
              uid 1111;
              class super-user;
              authentication {
                  encrypted-password "encrypted""; ## SECRET-DATA
                  ssh-rsa "ssh-rsa as;dklfja;slkfj;laskdjf;laksjdf;klasflkjas;ldfha;eiohfrqwoifhajsvn;asf;oha;wehf;ahf;klashd;flhas;ldhf;alshdfklhas;dfhalskdfh;lakshd;fklhasd;fhas;ldhfaklshdfklahsd;lfha;skldhf;laskhdf;klhasd;flhas;lvas;lvn;awepfoihweofhaosnv;lasnd;lfniowehfoavkljansd;lnas;dofhoaasjdflansd;fqwherf28yrp891h;oasv;akjsnv;asdf;ohasofhapowehfpauaohifoasidhf;lashdf;kashdf;ahsd;fhaposdhfpoasihdfoa user@domain"; ## SECRET-DATA
                  }
              }
          }
          services {
              netconf {
                  ssh;
                  }
              }
          }
    ```

* __[UPGRADE METHOD]__ For the "Juniper.junos_install_os" UPGRADE method, download TARGET and CURRENT JUNOS upgrade TGZ to __~/files/__ folder (or set symlink). Current JUNOS TGZ can be used for software rollback using the same process to upgrade since "request system rollback" no longer functions as of JUNOS 15.1.

* __[REBUILD METHOD]__ For the "nova rebuild" Reduild method, create a QCOW2 snapshot with your Target Code and preconfigure FXP0 for dhcp-client and user previously created for same access method (SSH-KEY or User/Pass). __PLEASE NOTE:__ The use of Ansible-Galaxy "OS_SERVER_ACTIONS" module may require a custom PY/PYC file (depending on 32-bit or 64-bit Linux OS).  These custom modules need to be used and are included in the REPO under the __~/files__ directory.

  - E.g.
    ```OS_SERVER_ACTIONS
    /usr/local/lib/python2.7/dist-packages/ansible/modules/cloud/openstack/
    ```

## Ansible Folder Structure

```FolderStructure
# ~/ == playbook directory

~/
    /files/
        junos-vsrx-15.1X49-D100.6-domestic.tgz
        junos-vsrx-15.1X49-D30.3-domestic.tgz
        junos-vsrx-15.1X49-D50.3-domestic.tgz
        junos-vsrx-15.1X49-D70.3-domestic.tgz
        mechid.pub
        makefile
        os_server_actions-23.py
        os_server_actions-23_32.pyc
        os_server_actions-23_64.pyc
    /log/
        /config/
        /debug/
        /output/
    /roles/
        /_template/
            /tasks/
                /_test/
                    00-testbed.c2x.yml
                    00-testbed.c3x.yml
                    00-testbed.yml
                /galaxy/
                    ...
                /junos/
                    ...
                /openstack/
                    ...
                main.yml
            /vars/
        /si_ha_active_active/
            /tasks/
                /galaxy/
                    galaxy-pause_30.yml
                    galaxy-pause_60.yml
                    galaxy-wait_for_netconf_30.yml
                    galaxy-wait_for_netconf_60.yml
                /junos/
                    junos-check_config.yml
                    junos-check_interfaces.yml
                    junos-check_license.yml
                    junos-check_rsi.yml
                    junos-delete_ais.yml
                    junos-dhcp_renew.yml
                    junos-disable_si_interfaces.yml
                    junos-enable_si_interfaces.yml
                    junos-get_config.yml
                    junos-get_facts.yml
                    junos-install_config.yml
                    junos-install_os.yml
                    junos-post_check_interfaces.yml
                    junos-post_checks.yml
                    junos-post_get_config.yml
                    junos-post_get_facts.yml
                    junos-reboot.yml
                /openstack/
                    os-server_rebuild.yml
                    os-vmi_down.c2x.yml
                    os-vmi_down.c3x.yml
                    os-vmi_up.c2x.yml
                    os-vmi_up.c3x.yml
                main.yml
            /vars/
                main.yml
                c2x.yml
                c3x.yml
        /si_ha_active_standby/
    /vars/
        vault.yml
    ansible.cfg
    hosts
    vsrx_c2x.rebuild.yml
    vsrx_c2x.upgrade.yml
    vsrx_c3x.rebuild.yml
    vsrx_c3x.upgrade.yml

```

## Ansible Modules Utilized

We will be utilizing native Ansible OpenStack modules (subset of Ansible Cloud Modules) and Juniper.junos modules:
* Ansible Cloud OpenStack & Networking JUNOS:
  >  - https://docs.ansible.com/ansible/list_of_cloud_modules.html#openstack
  >  - https://docs.ansible.com/ansible/os_port_module.html
  >  - https://docs.ansible.com/ansible/junos_command_module.html
  >  - https://docs.ansible.com/ansible/junos_config_module.html
* Juniper.junos:
  > - https://github.com/Juniper/ansible-junos-stdlib
  > - https://www.juniper.net/techpubs/en_US/release-independent/junos-ansible/information-products/pathway-pages/index.html
  > - https://junos-ansible-modules.readthedocs.io/en/1.4.0/
  > - https://junos-ansible-modules.readthedocs.io/en/1.4.0/junos_cli.html
  > - https://junos-ansible-modules.readthedocs.io/en/1.4.0/junos_get_config.html
  > - https://junos-ansible-modules.readthedocs.io/en/1.4.0/junos_install_os.html
  > - https://junos-ansible-modules.readthedocs.io/en/1.4.0/junos_shutdown.html
  > - https://junos-ansible-modules.readthedocs.io/en/1.4.0/junos_get_facts.html
  >
  > *** __PLEASE NOTE: the Juniper.junos role must be installed into the "ROLES" directory as specified in your ansible.cfg configuration file. If not specified, then this defaults to /etc/ansible/roles/__

## [VARIABLES] Inventory Hosts

__~/hosts__

The Hosts file will be used to feed variables ( {{ inventory_hostname }} & {{ ansible_host }} ) to Ansible Plays and Tasks for list of vSRX's to be upgraded. The {{ inventory_hostname }} will be used by the Ansible OpenStack Modules so this needs to be the name of the VM in OpenStack Nova.  The {{ ansible_host }} will be used by the Ansible Juniper.junos Modules and this will be the IP of the VM's FXP0 / Management Neutron Port.

We are investigating the possibility of automating the generation of the Hosts file using a modified version of the Dynamic Host Inventory script documented at:
> - https://docs.ansible.com/ansible/intro_dynamic_inventory.html#example-openstack-external-inventory-script
> - https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/openstack.py

## [VARIABLES] Other Variables Files

The main inputs for variables to be used by the Ansible Playbooks are from the following files:

```
INVENTORY HOST VARIABLES:
~/hosts
```
```
ENCRYPTED VARIABLES (Sensitive information like username/password/ssh-keys/etc.,):
~/vars/vault.yml
```
```
GENERIC ENVIRONMENT AND CODE VARIABLES:
~/roles/vsrx_si_haaa/vars/main.yml
```
```
CONTRAIL 2.x SPECIFIC VARIABLES:
~/roles/vsrx_si_haaa/vars/c2x.yml
```
```
CONTRAIL 3.x SPECIFIC VARIABLES
~/roles/vsrx_si_haaa/vars/c3x.yml
```

The above Variable Files need to be populated with information specific to your deployment site/LCP.

## Logs, Debug, Output, and Config
* ~/log/: Logs generally are Ansible logs for the Play or Task
* ~/log/config/: Config to be used after Rebuild
* ~/log/debug/: Debug are typically StdOut from each Task
* ~/log/output/: Output is the content generated from each Module used in the Tasks

## Roles
The roles previously used have been consolidated into a single vsrx_si_haaa role and will utilize TAGS to differentiate between Contrail 2.x and Rebuild/Upgrade methods.

## Executing the Ansible Playbook

###### Environment-Specific Variables & Hosts Required
* ~/hosts file needs to be populated with OS::NOVA::SERVER names of the vSRX VM's in OpenStack as well as the IP's as shown in the inventory hosts file example.
* Open Stack OpenRC variables set to specific LCP/Tenant/AuthURL/User for your environment

###### Syntax & Usage (Contrail 3.x Environment, HA-Active/Active, Rebuild)
```shell
#Eg.: ansible-playbook c3x-vsrx_si_haaa.yml --ask-vault-pass [-vvvv] --tags "c3x,rebuild" [--syntax-check]

ansibleserver:~/ansible-vsrx_upgrade$ ansible-playbook c3x-vsrx_si_haaa.yml --ask-vault-pass --tags "c3x,rebuild"
Vault password:
OpenStack User:
OpenStack Password:
PLAY [c3x-vsrx_si_haaa-playbook] ************************************************
TASK [setup] *******************************************************************
ok: [devsrxcnc001]
```

###### Syntax & Usage (Contrail 2.x Environment, HA-Active/Active, Rebuild)
```shell
#Eg.: ansible-playbook c2x-vsrx_si_haaa.yml --ask-vault-pass [-vvvv] --tags "c2x,rebuild" [--syntax-check]

ansibleserver:~/ansible-vsrx_upgrade$ ansible-playbook c2x-vsrx_si_haaa.yml --ask-vault-pass --tags "c2x,rebuild"
Vault password:
OpenStack User:
OpenStack Password:
PLAY [c2x-vsrx_si_haaa-playbook] ************************************************
TASK [setup] *******************************************************************
ok: [prdsrxcnc001]
```

###### Syntax & Usage (Contrail 3.x Environment, HA-Active/Active, Upgrade)
```shell
#Eg.: ansible-playbook c3x-vsrx_si_haaa.yml --ask-vault-pass [-vvvv] --tags "c3x,upgrade" [--syntax-check]

ansibleserver:~/ansible-vsrx_upgrade$ ansible-playbook c3x-vsrx_si_haaa.yml --ask-vault-pass --tags "c3x,upgrade"
Vault password:
OpenStack User:
OpenStack Password:
PLAY [c3x-vsrx_si_haaa-playbook] ************************************************
TASK [setup] *******************************************************************
ok: [devsrxcnc001]
```

###### Syntax & Usage (Contrail 2.x Environment, HA-Active/Active, Upgrade)
```shell
#Eg.: ansible-playbook c2x-vsrx_si_haaa.yml --ask-vault-pass [-vvvv] --tags "c2x,upgrade" [--syntax-check]

ansibleserver:~/ansible-vsrx_upgrade$ ansible-playbook c2x-vsrx_si_haaa.yml --ask-vault-pass --tags "c2x,upgrade"
Vault password:
OpenStack User:
OpenStack Password:
PLAY [c2x-vsrx_si_haaa-playbook] ************************************************
TASK [setup] *******************************************************************
ok: [prdsrxcnc001]
```

## Checklist:

- [ ] System and Software Requirements satisfied
- [ ] Open Stack Admin credentials for user executing playbook
- [ ] Able to execute Open Stack commands from CLI and connect to Controller Elements
- [ ] MechID for vSRX Upgrade with SSH Keys
- [ ] vSRX Image with MechID Admin User and SSH Keys configured
- [ ] vSRX Image with FXP0 set to DHCP and SSH Service Enabled
- [ ] Inventory Hosts file corectly populated
- [ ] ~/roles/si_ha_active_active/vars/main.yml has correct information
- [ ] ~/vars/vault.yml has correct Open Stack and vSRX login information

## References

> * https://www.ansible.com/blog/ansible-best-practices-essentials
> * https://docs.ansible.com/ansible/playbooks_best_practices.html
> * https://docs.ansible.com/ansible/playbooks_vault.html
> * https://docs.ansible.com/ansible/list_of_cloud_modules.html#openstack
> * https://www.juniper.net/techpubs/en_US/release-independent/junos-ansible/information-products/pathway-pages/index.html
> * https://github.com/Juniper/ansible-junos-stdlib
> * https://serversforhackers.com/an-ansible-tutorial
> * https://serversforhackers.com/series/ansible
> * http://www.ansiblebook.com/
****