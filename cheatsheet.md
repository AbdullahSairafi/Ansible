This document is written to provide a breif summary of how to use ansible based on cert guide rhce 8 book


# [Ansible Core Modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html)

# Environment Set Up
create 3 vms using rocky-linux os
- one vm as control node
- two vms as managed nodes
## VM prep (os, network, etc)
make sure all instances have two network profiles

1. NAT profile to access internet
2. lan profile to communicate with other two vms

Modify `/etc/hosts` file in the control node to easily refer to managed nodes

generate key-pair on control node and copy public key to managed nodes to utilize passwordless login to managed nodes

test connectivity amongst vms 

## Installing ansible

# [ad hoc commands](https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html)

ad hoc commands are used for running simple and quick tasks that are rarely repeated such as testing connectivity of server using `ping` module. 


`$ ansible <pattern> -m <module_name> -a "<module options>"`

example: starting `httpd` service on all *webservers* hosts

`$ ansible webservers -m service -a "name=httpd state=started"`

## finding ansible module information

to get information about a certain ansible module and see usage examples:

`$ ansible-doc <module_name>`

example: find information about the `service` module

`$ ansible-doc service`

to show a snippet of the module: (can be useful to quickly check required arguments)

`$ ansible-doc -s service`

## ad hoc commands with shell scripts

it is possible to run several ad hoc commands as a shell script

example:

```bash
#!/bin/bash

ansible all -m yum "name=httpd state=latest"
ansible all -m service "name=httpd state=started enabled=yes"
```

**Note**: make sure to make the script executable using `chmod` command

---

# [Intro to playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html)

ansible playbooks are the core of ansible where you can write yaml scripts to manage multiple target servers. playbooks are suitable for configuration management. 

```yaml
---
# [play 1]
- name: install start and enable httpd # name of the play
  hosts: all # targeted hosts
  tasks: # list of tasks to be executed
    - name: install package # task name (optional but recommended to include)
      yum: # name of the module to be executed
        # the following are module options
        name: httpd
        state: installed
    - name: start and enable service # task name
      service: # name of the module to be executed
        # module options
        name: httpd
        state: started
        enabled: yes
...
```
to execute the playbook

`ansible-playbook <playbook>`

**Tip**: To make working with indentation easier, add the following to `~/.vimrc` file

`autocmd FileType yaml setlocal ai ts=2 sw=2 et`
## [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html#yaml-syntax)

**Tip**: to make sure syntax is correct, you can run the command

`ansible-playbook --syntax-check <playbook>`

## multi-play playbooks

benefits of multiplay playbooks:

- define different group of hosts
- different connection parameters (user, privilege esclation, etc)
  
**Tip**: It is best practice to develop small and specific playbooks than larger playbooks. 

- code/script reusability using includes
- easier for troubleshooting
- flexibility for wide range of configuration management

Example of multiplay playbook

1. Play 1: install, start, and enable `httpd`
2. Play 2: test connectivity from **control** node to managed node **ansible1** 
```yaml
---
# [play 1]
- name: install start and enable httpd
  hosts: all
  tasks:
    - name: install package
      yum:
        name: httpd
        state: latest
    - name: start and enable service
      service:
        name: httpd
        state: started
        enabled: yes
# [play 2]
- name: test httpd accessibility
  hosts: localhost
  tasks:
  - name: test httpd access
    uri:
      url: http://ansible1
```

**Note**: The above playbook will generate an error because we have not opened the firewall on ansible1 node. The following playbook fixes the error.

```yaml
---
# [play 1]
- name: enable web server
  hosts: ansible1
  tasks:
    - name: install httpd and firewalld
      yum:
        name:
          - httpd
          - firewalld

    - name: create a welcome page (optional)
      copy:
        content: "welcome to the webserver\n"
        dest: /var/www/html/index.html

    - name: start and enable webserver
      service:
        name: httpd
        state: started
        enabled: true

    - name: enable firewall
      service:
        name: firewalld
        state: started
        enabled: true

    - name: open http service in firewall
      firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: yes

# [play 2]
- name: test webserver access
  hosts: localhost
  become: no
  tasks:

    - name: test webserver access
      uri:
        url: http://ansible1
        return_content: yes
        status_code: 200
```

---

# Ansible Facts and Variables

| Variable Type | Use |
| :--- | :--- |
| `Fact` | Variables that contain values describing specific system properties|
| `Variable` | Variables that are defined by the user |
| `Magic Variable` | System variable that is automatically set |

## [Ansible Facts](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html#ansible-facts)

Ansible facts are system properties that are collected
when Ansible executes on a remote system. After facts
are gathered, the facts can be used as variables

```yaml
---
- name: show fact gathering
  hosts: all
  tasks:
    - name: show all facts
      debug:
        var: ansible_facts
...
```
The above playbook will return all the ansible facts of all managed hosts.

Common ansible facts:

| Variable Type | Use |
| :--- | :--- |
| `ansible_facts["hostname"]` | Hostname |
| `ansible_facts["distribution"]` | Linux distribution |
| `ansible_facts["default_ipv4"]["address"]` | Main IPv4 address |
| `ansible_facts["interfaces"]` | List of network interfaces |
| `ansible_facts["devices"]` | List of attached sotrage devices |
| `ansible_facts["distribution_version"]` | Current distribution version |

```yaml
---
- hosts: all
  name: gather IPv4 addresses from all hosts
  tasks:
    - name: show IP address
      debug:
        msg: >
          {{ ansible_facts["hostname"] }} has IP addresses: {{ ansible_facts["all_ipv4_addresses"] }}
...
```

## [Variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#playbooks-variables)

variables are placeholders for dynamic data and are used to make playbooks re-usable and work with different systems. Variables come in handy especially when working with conditionals and loops.

### Variables in a play

```yaml
---
- name: installing httpd using variables
  hosts: all
  vars: # variable defined at play level
    httpd_package: httpd
  tasks:
    - name: install package
      yum:
        name: "{{ httpd_package }}" # note: variables is placed in quotes
        state: latest
...
```
**Note**: when referencing a variable by itself with no text expression wrapped around it, then it must be placed in quotes like `"{{ httpd_package }}"` to create a correct YAML syntax.


### Variables in included files

play variables are suitable for smaller playbooks, but as more variables are used, it is better to define variables in a seperate file and include the variable file in the playbook.

```yaml
---
- name: using a variable include file
  hosts: all
  vars_files: ../vars/packages.yaml
  tasks:
    - name: install packages
      yum:
        name: "{{ my_packages }}"
        state: latest
...
```

and variable file `vars/packages.yaml`:

```yaml
---
my_packages:
  - httpd
  - mysql
  - firewalld
...
```

**Note**: ensure the correct file path when including variable files. 

<pre><font color="#0087FF">.</font>
├── ansible.cfg
├── inventory.yaml
├── <font color="#0087FF">playbooks</font>
│   └── ansible_facts.yaml
└── <font color="#0087FF">vars</font>
    └── packages.yaml

2 directories, 4 files</pre>

command to run playbook:

<pre>$ ansible-playbook playbooks/external_vars.yaml</pre>



**Note**: `yum` module supports lists and that is why we are able to use it to install multiple packages at once. some modules do not support lists and in that case, the best option is to use loops.

### [Magic Variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html#information-about-ansible-magic-variables)

Magic variables are variables set by ansible to reflect ansible internal state such as python version, hosts and groups in inventory, and directories for playbooks and roles.

Table showing common magic variables

| Variable  | Use |
| :--- | :--- |
| `hostvars` | all hosts in inventory and their assigned variables |
| `groups` | contains all groups in inventory |
| `group_names` | lists groups this host is currently a member of |
| `inventory_hostname` | specifies the inventory host name for the current host |
| `inventory_file` | specifies the currently used inventory file |


## [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)

Ansible Vault is used to encrypt and decrypt sensitve data that should not be stored as plain text such as passwords

Ansible Vault works as following:

1. Sensitive data is stored as variable in a seprate yaml file
2. the yaml file is encrypted using the `ansible-vault` command and the file is assigned a password
3. The assigned password is used to decrypt the encrypted yaml file to refernce the variable in a playbook or command line. 


### Creating an encrypted file

<pre>$ ansible-vault create secret.yaml
New Vault password: 
Confirm New Vault password: 
</pre>


<pre>$ ansible-vault view secret.yaml
Vault password: 
username: aiden
password: password
</pre>

Note: if `secret.yaml` is viewed using `cat` or using a text editor, we would see an encrypted value and not the same output as above

Now lets put everything together and create a user and on a managed host and set its password using ansible vault:

```yaml
---
- name: Create user with password using vault
  hosts: ansible3
  vars_files: 
    - ../vars/secret.yaml 
  tasks:
    - name: create user
      user:
        name: "{{ username }}"
        password: " {{ password | password_hash('sha512') }}"
...
```

<pre><font color="#0087FF">.</font>
├── ansible.cfg
├── inventory.yaml
├── <font color="#0087FF">playbooks</font>
│   ├── ansible_facts.yaml
│   └── *create_user.yaml*
└── <font color="#0087FF">vars</font>
    ├── packages.yaml
    └── *secret.yaml*
</pre>

<pre>$ ansible-playbook --ask-vault-pass playbooks/create_user.yaml</pre>


**Note**: this produces a warning because the password is unhashed when it is given to the user module. we will look into this later

**Note**: adding `| password_hash('sha512')` removed the warning but when loggin in as the newly created user, there is authentication failure

Common ansible vault options


| Option  | Use |
| :--- | :--- |
| `create` | create new encrypted file |
| `encrypt` | encrypt an existing file |
| `encrypt_string` | encrypt a sting |
| `decrypt` | decrypt an existing file |
| `rekey` | changes the password on encrypted file |
| `view` | view content of encrypted file |
| `edit` | edit encrypted file |


---

# [Loops](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html)

loops come in handy when working with modules that do not accept lists. Loops will be used to iterate over a list that is ususally stored as a variable.

```yaml
---
- name: install packages and start services using loops
  hosts: all
  vars:
    packages:
      - httpd
      - mysql-server
      - vsftpd
      
    services:
      - httpd
      - mysqld
      - vsftpd
  tasks:
    - name: install packages
      yum: 
        name: "{{ packages }}"
        state: latest

    - name: start services
      service:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop: "{{ services }}"
...
```

**Note**: if we are looping over a multivalued variables (object-like variables) then we can access attributes using dot notation. For example, if we have a variable with attributes `username` and `password`, then we would access them using `item.username` and `item.password` 


TODO: add example to use register (stores task output for logging)

# [Conditionals](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#playbooks-conditionals)

```yaml
---
- name: conditional install
  hosts: all
  tasks:
    - name: install apache on Red Hat and family
      package:
        name: httpd
        state: latest
      when: ansible_facts[’os_family’] == "RedHat"
    - name: install apache on Ubuntu and family
      package:
        name: apache2
        state: latest
      when: ansible_facts[’os_family’] == "Debian"
...
```

Conditional tests table:


| Conditional Test  | Example |
| :--- | :--- |
| variable exists | `<variable> is defined` |
| variable does not exist | `<variable> is not defined` |
| variable exists in a list | `<variable> in <list>` |
| variable is true | `<variable>` |
| variable is false | `not <variable>` |
| equal (numeric \| string) | `key == value` or `key == "value"` |
| not equal | `key != value` |
| less than | `key < value` |
| less than or equal to | `key <= value` |
| greater than | `key > value` |
| greater than or equal to | `key >= value` |

### Multiple conditions
```yaml
---
- name: using multiple conditions
  hosts: all
  tasks:
    - package:
        name: httpd
        state: removed
      when: >
      ( ansible_facts[’distribution’] == "RedHat" and
      ansible_facts[’memfree_mb’] < 512 )
      or
      ( ansible_facts[’distribution’] == "CentOS" and
      ansible_facts[’memfree_mb’] < 256 )
...
```

### Conditions with loops

```yaml
tasks:
    - name: Run with items greater than 5
      command: echo {{ item }}
      loop: [ 0, 2, 4, 6, 8, 10 ]
      when: item > 5
```

```yaml
---
- name: test register
  hosts: all
  tasks:
    - shell: cat /etc/passwd
      register: passwd_contents # store the content in a varaiable
    - debug:
        msg: passwd contains user lisa
      when: passwd_contents.stdout.find(’lisa’) != -1
...
```


# [Handlers](https://docs.ansible.com/ansible/latest/user_guide/playbooks_handlers.html)


Handlers are tasks that run only when notified

```yaml
---
- name: Verify apache installation
  hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
    - name: Ensure apache is at the latest version
      yum:
        name: httpd
        state: latest

    - name: Write the apache config file
      template:
        src: /srv/httpd.j2
        dest: /etc/httpd.conf
      notify:
      - Restart apache

    - name: Ensure apache is running
      service:
        name: httpd
        state: started
        enabled: yes

  handlers:
    - name: Restart apache
      service:
        name: httpd
        state: restarted
...
```

Handlers can listen on topics that can group multiple handlers as follows:

```yaml
tasks:
  - name: Restart everything
    command: echo "this task will restart the web services"
    notify: "restart web services"

handlers:
  - name: Restart memcached
    service:
      name: memcached
      state: restarted
    listen: "restart web services"

  - name: Restart apache
    service:
      name: apache
      state: restarted
    listen: "restart web services"
...
```

# [Block](https://docs.ansible.com/ansible/latest/user_guide/playbooks_blocks.html)

```yaml
  tasks:
    - name: Attempt and graceful roll back demo
      block:
        - name: Print a message
          debug:
            msg: 'I execute normally'

        - name: Force a failure
          command: /bin/false

        - name: Never print this
          debug:
            msg: 'I never execute, due to the above task failing, :-('

      rescue:
        - name: Print when errors
          debug:
            msg: 'I caught an error'

        - name: Force a failure in middle of recovery! >:-)
          command: /bin/false

        - name: Never print this
          debug:
            msg: 'I also never execute :-('

      always:
        - name: Always do this
          debug:
            msg: "This always executes"
...
```

## To explore later

- ignore_errors
- failed_when
- force_handlers
- changed_when

# Managing Files

## Managing File Attributes

```yaml
---
- name: stat module tests
  hosts: ansible1
  tasks:
    - command: touch /tmp/statfile
    
    - stat:
        path: /tmp/statfile
      register: st

    - name: show current values
      debug:
        msg: current value of the st variable is {{ st }}
    
    - name: changing file permissions if that’s needed
      file:
        path: /tmp/statfile
        mode: 0640
      when: st.stat.mode != ’0640’
...
```

## Managing File Contents

```yaml
---
- name: configuring SSH
  hosts: all
  tasks:
    - name: disable root SSH login
      lineinfile:
        dest: /etc/ssh/sshd_config
        regexp: "^PermitRootLogin"
        line: "PermitRootLogin no"
      notify: restart sshd

  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted
...
```

```yaml
---
- name: modifying block in a file
  hosts: all
  tasks: 
    - name: touch file
      file:
        path: /tmp/hosts
        state: touch
    
    - name: add block text to file
      blockinfile:
        path: /tmp/hosts
        block: | # insert as multiple lines
          192.168.4.110 host1.exmaple.com
          192.168.4.120 host2.example.com
        state: present
...
```

Common File Modules:
- `copy`
- `fetch`
- `file`
- `acl`
- `find`
- `lineinfile`
- `blockinfile`
- `replace`
- `synchronize`
- `stat`


# Managing SELinux

| Module | Use |
| :--- | :--- |
| `file` | manages context on files but not in selinux policy |
| `sefcontext` | manages file context in the selinux policy |
| `command` | used to run `restorecon` command after `sefcontext` |
| `selinux` | manages current selinux state |
| `seboolean` | manages selinux booleans  |


```yaml
---
- name: show selinux
  hosts: all
  tasks:
    - name: install required pacakges
      yum:
        name: policycoreutils-python-utils
        state: present
    - name: create testfile
      file:
        name: /tmp/selinux
        state: touch
    - name: set selinux context
      sefcontext:
        target: /tmp/selinux
        setype: httpd_sys_content_t
        state: present
      notify:
        - run restorecon

  handlers:
    - name: run restorecon
      command: restorecon -v /tmp/selinux 
...
```

```yaml
---
- name: Managing web server SELinux properties
  hosts: ansible1
  tasks:
    - name: ensure SELinux is enabled and enforcing
      selinux:
        policy: targeted
        state: enforcing

    - name: install the webserver
      yum:
        name: httpd
        state: latest

    - name: open the firewall service
      firewalld:
        service: http
        immediate: yes
        permanent: yes
        state: enabled

    - name: create the /web directory
      file:
        name: /web
        state: directory

    - name: create the index.html file in /web
      copy:
        content: 'welcome to the exercise82 web server'
        dest: /web/index.html
        
    - name: use lineinfile to change webserver configuration
      lineinfile:
        path: /etc/httpd/conf/httpd.conf
        regexp: '^DocumentRoot "/var/www/html"'
        line: DocumentRoot "/web"

    - name: use lineinfile to change webserver security
      lineinfile:
        path: /etc/httpd/conf/httpd.conf
        regexp: '^<Directory "/var/www">'
        line: '<Directory "/web">'
      notify: start webserver

    - name: use sefcontext to set context on new documentroot
      sefcontext: 
        target: '/web(/.*)?'
        setype: httpd_sys_content_t
        state: present

    - name: run the restorecon command
      command: restorecon -Rv /web

    - name: allow the web server to run user content
      seboolean:
        name: httpd_read_user_content
        state: yes
        persistent: yes

  handlers:
    - name: start webserver
      service:
        name: httpd
        status: started
        enabled: yes

...
```

# [Jinja2 Templates](https://jinja.palletsprojects.com/en/3.1.x/templates/)

Jinja2 is used to create templates which in turn are used to generate configuration files on managed hosts. 

```jinja
{# basic syntax #}

Hostname: {{ ansible_facts.hostname }}
IPv4 Addresses: {{ ansible_facts.all_ipv4_addresses }}
OS Family: {{ ansible_facts.os_family }}
OS Distribution: {{ ansible_facts.distribution }}

variable from playbook: {{ myVar }}

{# conditions #}
{% set mem_total = ansible_facts.memtotal_mb %}
Total Memory: {{ mem_total }} MB

{% if mem_total > 700 %}
lots of memory
{% elif mem_total < 700 and mem_total > 200%}
good enough memory
{% else %}
memory is low
{% endif %}

{# loops #} {# simple copy of /etc/hosts file #}
{% for host in groups['all']%}
{{ hostvars[host]['ansible_facts']['default_ipv4']['address'] }} {{ hostvars[host]['ansible_facts']['fqdn'] }} {{ hostvars[host]['ansible_facts']['hostname'] }}
{% endfor %}
```

playbook that uses the jijna template above to generate the text file on target machines (based on inventory file)

```yaml
---
- name: template file example
  hosts: all
  vars:
    - myVar: hello world
  tasks:
    - name: use a template to create a text file
      template:
        src: ../templates/example.j2
        dest: /tmp/result.txt # found on target machines
...
```

files and directories structure

<pre><font color="#0087FF">.</font>
├── ansible.cfg
├── inventory.yaml
├── <font color="#0087FF">playbooks</font>
│   └── template.yaml
├── <font color="#0087FF">templates</font>
    └── example.j2
</pre>

**Note**: if not all targets are reachable, ansible might generate an error regarding dict attributes for `ansible_facts` which might be confusing since proper syntax is used. Just remember to make all target reachable or include `ignore_errors` flag in the playbook tasks


---

# [Roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html#)


---

# [Large Environments]

## imports and includs

import and includ tasks, playbook, files, etc


# Managing Software

| Module | Use |
| :--- | :--- |
| `yum` | manages software packages on RHEL and CentOS |
| `apt` | manages software on Ubuntu |
| `dnf` | manages packages on Fedora |
| `package` | manages packages on any linux distribution |
| `yum_repository` | manages repositories |

## [Configuring repository access](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_repository_module.html)

```yaml
---
- name: setting up repo access
- hosts: all
- tasks:
  
  - name: set up example repo
    yum_repository:
      name: example repo
      description: rhce8 example repo
      file: examplerepo
      baseurl: ftp://control.example.com/repo/
      enabled: yes
      gpgcheck: no
...
```

## Package Facts

package facts must be directly invoked in a playbook to get info about installed packages on a managed nodes. Let us see an example

```yaml
---
- name: playbook to get package facts
- hosts: all
- vars:
  - my_package: httpd
- tasks:

  - name: install package 
    package:
      name: "{{ my_package }}"
      state: latest

  - name: use package facts module to collect info
    package_facts:
      manager: auto
  
  - name: show package facts for {{ my_package }}
    debug:
      var: ansible_facts["packages"][my_package]
    when: my_package in ansible_facts["packages"]
...
```

## Managing RHEL Subscription

```yaml
---
- name: use subscription manager to register and set up repos
  hosts: ansible5
  tasks:
  
    - name: register and subscribe ansible5
      redhat_subscription:
        username: bob@example.com
        password: verysecretpassword
        state: present

    - name: configure additional repo access
      rhsm_repository:
        name:
          - rh-gluster-3-client-for-rhel-8-x86_64-rpms
          - rhel-8-for-x86_64-appstream-debug-rpms
        state: present
```

More advanced example for managing software

```yaml
---
- name: add host to inventory
  hosts: localhost
  tasks:
    - fail: # must pass variables newhost and newhostip using -e option
        msg: "add the options -e newhost=hostname -e newhostip=ip.ad.dr.ess and try again"
      when: (newhost is undefined) or (newhostip is undefined)
    - name: add new host to inventory
      lineinfile:
        path: inventory
        state: present
        line: "{{ newhost }}"
    - name: add new host to /etc/hosts
      lineinfile:
        path: /etc/hosts
        state: present
        line: "{{ newhostip }} {{ newhost }}"
    tags: addhost

- name: configure a new RHEL host
  hosts: "{{ newhost }}"
  remote_user: root
  become: false
  tasks:
    - name: configure user ansible
      user:
        name: ansible
        groups: wheel
        append: yes
        state: present
    - name: set a password for this user
      shell: 'echo password | passwd --stdin ansible'
    - name: enable sudo without passwords
      lineinfile:
        path: /etc/sudoers
        regexp: '^%wheel'
        line: '%wheel ALL=(ALL)  NOPASSWD: ALL'
        validate: /usr/sbin/visudo -cf %s
    - name: create SSH directory in user ansible home
      file:
        path: /home/ansible/.ssh
        state: directory
        owner: ansible
        group: ansible
    - name: copy SSH public key to remote host
      copy:
        src: /home/ansible/.ssh/id_rsa.pub
        dest: /home/ansible/.ssh/authorized_keys
    tags: setuphost

- name: use subscription manager to register and set up repos
  hosts: "{{ newhost }}"
  vars_files:
    - exercise123-vault.yaml
  tasks:
    - name: register and subscribe {{ newhost }}
      redhat_subscription:
        username: "{{ rhsm_user }}"
        password: "{{ rhsm_password }}"
        state: present
    - name: configure additional repo access
      rhsm_repository:
        name: 
        - rh-gluster-3-client-for-rhel-8-x86_64-rpms
        - rhel-8-for-x86_64-appstream-debug-rpms 
        state: present
    tags: registerhost
```

**Note**: we will improve on this example in the following areas

- setting up passwords to new users
- copying keypairs