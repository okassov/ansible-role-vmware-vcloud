# Ansible role for VMWare vCloud Director

This role provision Virtual Machnines using VMWare vCloud Director (Infrastructure as Code)

Optional:

- External Disk (Named Disk)
- Additional Network Interface Card (NIC)

## Requirements

- pyvcloud>=23.0.2
- lxml
- requests

## Variables

```yaml
check_mode: yes
```

## Usage

Example of virtual-machines.yaml for provision multiple VMs:

```yaml
---

vcd_virtual_machines:
  - {
      vm_name: "example-vm-1",
      vm_ip: "192.168.1.100",
      vm_netmask: "/24",
      vm_gateway: "192.168.1.1",
      vm_dns: "1.1.1.1",
      vapp: "test",
      state: "present",
      power_on: "true",
      virtual_cpus: 2
      cores_per_socket: 2
      memory: 4096
      ip_allocation_mode: "MANUAL",
      network: "test",
      storage_profile: "",
    }
  - {
      vm_name: "example-vm-2",
      vm_ip: "192.168.1.102",
      vm_netmask: "/24",
      vm_gateway: "192.168.1.1",
      vm_dns: "1.1.1.1",
      vapp: "test",
      state: "present",
      power_on: "true",
      virtual_cpus: 2
      cores_per_socket: 2
      memory: 4096
      ip_allocation_mode: "MANUAL",
      network: "test",
      storage_profile: "",
    }
```

Example of virtual-machines.yaml for provision VM with external Data Disk:

```yaml
---

vcd_virtual_machines:
  - {
      vm_name: "example-vm-1",
      vm_ip: "192.168.1.100",
      vm_netmask: "/24",
      vm_gateway: "192.168.1.1",
      vm_dns: "1.1.1.1",
      vapp: "test",
      state: "present",
      power_on: "true",
      virtual_cpus: 2
      cores_per_socket: 2
      memory: 4096
      ip_allocation_mode: "MANUAL",
      network: "test",
      storage_profile: "",
      disk: {
        size: 100, # Size of external data disk in MB
        storage_profile: "qc30-ds-hpe-platinum02-policy",
        controller: lsilogic,
        state: "present"
      },
    }
```

Example of virtual-machines.yaml for provision VM with additional NIC:

```yaml
---

vcd_virtual_machines:
  - {
      vm_name: "example-vm-1",
      vm_ip: "192.168.1.100",
      vm_netmask: "/24",
      vm_gateway: "192.168.1.1",
      vm_dns: "1.1.1.1",
      vapp: "test",
      state: "present",
      power_on: "true",
      virtual_cpus: 2
      cores_per_socket: 2
      memory: 4096
      ip_allocation_mode: "MANUAL",
      network: "test",
      storage_profile: "",
      nic: {
        network: "test",
        state: "present"
      }
    }
```

Example of [init-script.yaml](examples/init-script.yaml) for post-customization then provision VM:

```yaml
---

initial_script: |
  #!/bin/bash
  # Maintainer: Marat Okassov
  # Inital script for configure a Ubuntu Server guest operating system.

  ### Configure network
  sudo cat<<EOF > /etc/netplan/00-installer-config.yaml
  network:
    version: 2
    renderer: networkd
    ethernets:
      $(ip add | grep 2: | awk '{print$2}' | tr -d ':'):
        dhcp4: false
        dhcp6: false
        addresses:
          - {{ item.vm_ip }}{{ item.vm_netmask }}
        gateway4: {{ item.vm_gateway }}
        nameservers:
          addresses:
            - {{ item.vm_dns }}
            - 8.8.8.8
  EOF

  ### Restart network
  sudo netplan apply
``` 

Example of [playbook.yaml](examples/playbook.yaml) for provision VMs:

```yaml
---

- name: vCloudDirectorAnsible
  hosts: localhost
  environment:
    env_user: "{{ lookup('env', 'ANSIBLE_VAR_vcd_user') }}"
    env_password: "{{ lookup('env', 'ANSIBLE_VAR_vcd_password') }}"
    env_host: "https://qc30-cloud.qazcloud.kz/"
    env_org: "KT-B2C"
    #env_api_version: 33.0
    env_verify_ssl_certs: true
  vars_files:
    - ../env-global.yaml
    - init-script.yaml
    - virtual-machines.yaml
  vars:
    vcd_vm_template_name: "test-packer-template"
    vcd_vm_template_vm_name: "test-packer"
    vcd_vm_deploy_vdc: "vdc-KT-B2C"
    vcd_vm_root_password: "root"
    app: "mongodb"
  roles:
    - role: vmware-vcloud
```

Example of infrastructure repo:

```
example-project
|___dev/
|   |___env-global.yaml
|   |___mongodb/
|   |   |___playbook.yaml
|   |   |___init-script.yaml
|   |   |___virtual-machines.yaml
|   |   |___requirements.yaml
|   |   |___requirements.txt
|   |   |___roles/
|   |   |   |___vmware-vcloud/
|   |   |       |___ ...
|   |   |___ansible/
|   |       |___playbook.yaml
|   |       |___inventory.yaml
|   |       |___requirements.yaml
|   |       |___requirements.txt
|   |       |___roles/
|   |           |___mongodb
|   |___postgresql/
|       |___playbook.yaml
|       |___init-script.yaml
|       |___virtual-machines.yaml
|       |___requirements.yaml
|       |___requirements.txt
|       |___roles/
|       |   |___vmware-vcloud/
|       |       |___ ...
|       |___ansible/
|           |___playbook.yaml
|           |___inventory.yaml
|           |___requirements.yaml
|           |___requirements.txt
|           |___roles/
|               |___postgresql
|___prod/
    |___env-global.yaml
    |___mongodb/
    |   |___playbook.yaml
    |   |___init-script.yaml
    |   |___virtual-machines.yaml
    |   |___requirements.yaml
    |   |___requirements.txt
    |   |___roles/
    |   |   |___vmware-vcloud/
    |   |       |___ ...
    |   |___ansible/
    |       |___playbook.yaml
    |       |___inventory.yaml
    |       |___requirements.yaml
    |       |___requirements.txt
    |       |___roles/
    |           |___mongodb
    |___postgresql/
        |___playbook.yaml
        |___init-script.yaml
        |___virtual-machines.yaml
        |___requirements.yaml
        |___requirements.txt
        |___roles/
        |   |___vmware-vcloud/
        |       |___ ...
        |___ansible/
            |___playbook.yaml
            |___inventory.yaml
            |___requirements.yaml
            |___requirements.txt
            |___roles/
                |___postgresql
```

Basic steps for provision infra:

Prepare:
```console
cd dev/mongodb/
python3 -m venv venv
source venv/bin/activate
pip3 install --upgrade pip
pip3 install -r requirements.txt
```

Change VM parameters in virtual-machines.yaml

Download and provision VM **without additional NIC**:
```console
ansible-galaxy install -r requirements.yaml -p roles/
export ANSIBLE_VAR_vcd_user="admin"
export ANSIBLE_VAR_vcd_password="changeme"
ansible-playbook playbook.yaml
```

Download and provision VM **with additional NIC** first time:
```console
ansible-galaxy install -r requirements.yaml -p roles/
export ANSIBLE_VAR_vcd_user="admin"
export ANSIBLE_VAR_vcd_password="changeme"
ansible-playbook playbook.yaml -e check_mode=false # Only once the provision VM you need check_mode=false for idempotence
```

## Docs

- [Module configuration and resources example](https://github.com/vmware/ansible-module-vcloud-director/blob/master/docs/index.md)

## License

MIT
