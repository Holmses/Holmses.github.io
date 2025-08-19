# Ansible_ssh_playbook

```yaml
---
- hosts: all
  gather_facts: no
         
  tasks:
  - name: install ssh key
    authorized_key: 
      user: root2020
      key: "{{ lookup('file','/root/.ssh/id_rsa.pub')}}"
      state: present
```

/etc/ansible/ansible.cfg

添加公钥不验证

```bash
# Since Ansible 2.12 (core):
# To generate an example config file (a "disabled" one with all default settings, commented out):
#               $ ansible-config init --disabled > ansible.cfg
#
# Also you can now have a more complete file by including existing plugins:
# ansible-config init --disabled -t all > ansible.cfg

# For previous versions of Ansible you can check for examples in the 'stable' branches of each version
# Note that this file was always incomplete  and lagging changes to configuration settings

# for example, for 2.9: https://github.com/ansible/ansible/blob/stable-2.9/examples/ansible.cfg

[defaults]
host_key_checking = False
```