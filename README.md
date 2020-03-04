Sudo prompt timeout on redhat hosts only 
========================================

**This was not an isssue with ansible**
The problem was some script in `/etc/profile.d` throwing terminal escape sequences to the stdout which ansible dindn't handle well. This maybe could be fixed on the side of ansible but is probably not needed since such script should only run on the condition that the stdout is a terminal (and in my case if it is a serial console).
After adding that condition everything works fine again.

The bug report that I submited about this is https://github.com/ansible/ansible/issues/67969, which I have closed.

I will also move this repo to archive.


Description
-----------

Sudo times out waiting for prompt when become password is asked for. This situation only seems to occur when the host in question is a RedHat based system. If pipelining is anabled the issue disappears, and all tested hosts work again.

Control machine
---------------

VM running CentOS 8, ansible installed from EPEL.

**OS**
```
# cat /etc/os-release 
NAME="CentOS Linux"
VERSION="8 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="8"
PLATFORM_ID="platform:el8"
PRETTY_NAME="CentOS Linux 8 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:8"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-8"
CENTOS_MANTISBT_PROJECT_VERSION="8"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="8"

# cat /etc/centos-release
CentOS Linux release 8.1.1911 (Core) 
```

**Ansible**
```
$ ansible --version
ansible 2.9.3
  config file = /home/mark/.ansible.cfg
  configured module search path = ['/home/mark/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.6/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.6.8 (default, Nov 21 2019, 19:31:34) [GCC 8.3.1 20190507 (Red Hat 8.3.1-4)]
```

**ansible.cfg**
```
[defaults]
inventory=inventory
host_key_checking=no
deprecation_warnings=no
```

**inventory**
```
[testservers]
centos8
centos7
rhel8
fedora31
debian
ubuntu16-04
ubuntu18-04
```

Managed hosts
-------------

They are all VMs whit default installations of the distribution they're named after. (debian is debian stable, currently 10)
For testing this bug a user was created named `sudoall` which has ssh-keys installed from the control host. The user has the following sudo policy:
```
sudoall    ALL=(ALL) ALL
```

Playbooks
---------

**There are two playbooks I tested with the first one is the following**
```
$ cat test.yml
# Test sudo bug
#
# vim:sw=2:sts=2:et
---
- name: 'Test sudo bug'
  gather_facts: no
  hosts: testservers
  user: sudoall
  become_method: sudo
  become_user: root
  tasks:
    - name: 'Run id'
      command: id
      become: yes
      register: id_result

    - name: 'Show id stdout'
      debug:
        var: id_result.stdout
```
It should run `id` on all the servers and show the stdout. It requires to be run with `-K` since user `sudoall` requires a password for sudo.

*Result*:
```
$ ansible-playbook -K test.yml 
BECOME password: 

PLAY [Test sudo bug] ********************************************************************************

TASK [Run id] ***************************************************************************************
[WARNING]: Platform linux on host debian is using the discovered Python interpreter at
/usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more
information.

changed: [debian]
changed: [ubuntu16-04]
changed: [ubuntu18-04]
fatal: [centos7]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt: "}
fatal: [fedora31]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt: "}
fatal: [rhel8]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt: "}
fatal: [centos8]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt: "}

TASK [Show id stdout] *******************************************************************************
ok: [ubuntu16-04] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root)"
}
ok: [debian] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root)"
}
ok: [ubuntu18-04] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root)"
}

PLAY RECAP ******************************************************************************************
centos7                    : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
centos8                    : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
debian                     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
fedora31                   : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
rhel8                      : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
ubuntu16-04                : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ubuntu18-04                : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
For some reason it failes on all RedHat based systems (RHEL, CentOS and Fedora) but not on Debian based (Debian and Ubuntu).
Full debug log `test.log` obtained by `env ANSIBLE_LOG_PATH=test.log ANSIBLE_DEBUG=1 ansible-playbook -vvvv -K test.yml`.
Debug log for only CentOS 8 host `test_centos8.log` obtained by `env ANSIBLE_LOG_PATH=test_centos8.log ANSIBLE_DEBUG=1 ansible-playbook -vvvv -K test.yml --limit centos8`.
The debug output seems to show the prompt is seen on the stdout
```
11936 1583235011.48993: stdout chunk (state=0):
>>>[sudo via ansible, key=euxtwlkwnsbduyzuqhrndztatxspjtpq] password:<<<

```

**The second playbook looks like the following**
```
# Test sudo bug
#
# vim:sw=2:sts=2:et
---
- name: 'Test sudo bug'
  gather_facts: no
  hosts: testservers
  user: sudoall
  become_method: sudo
  become_user: root
  vars:
    ansible_ssh_pipelining: yes
  tasks:
    - name: 'Run id'
      command: id
      become: yes
      register: id_result

    - name: 'Show id stdout'
      debug:
        var: id_result.stdout
```
The same thing but with pilelining enabled

*Result:*
```
$ ansible-playbook pipeline.yml -K
BECOME password: 

PLAY [Test sudo bug] ********************************************************************************

TASK [Run id] ***************************************************************************************
[WARNING]: Platform linux on host debian is using the discovered Python interpreter at
/usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more
information.

changed: [debian]
changed: [centos7]
changed: [centos8]
changed: [rhel8]
changed: [fedora31]
changed: [ubuntu16-04]
changed: [ubuntu18-04]

TASK [Show id stdout] *******************************************************************************
ok: [centos8] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023"
}
ok: [rhel8] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023"
}
ok: [centos7] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023"
}
ok: [fedora31] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023"
}
ok: [debian] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root)"
}
ok: [ubuntu16-04] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root)"
}
ok: [ubuntu18-04] => {
    "id_result.stdout": "uid=0(root) gid=0(root) groups=0(root)"
}

PLAY RECAP ******************************************************************************************
centos7                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centos8                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
debian                     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
fedora31                   : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
rhel8                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ubuntu16-04                : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ubuntu18-04                : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```
Full debug output in `pipeline.log` obtained with `env ANSIBLE_LOG_PATH=pipeline.log ANSIBLE_DEBUG=1 ansible-playbook -vvvv -K pipeline.yml`.
Debug output for CentOS 8 only `pipeline_centos8.log` obtained with `env ANSIBLE_LOG_PATH=pipeline_centos8.log ANSIBLE_DEBUG=1 ansible-playbook -vvvv -K pipeline.yml --limit centos8`
