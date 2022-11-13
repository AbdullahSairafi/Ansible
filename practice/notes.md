Hints and notes for practice exam from [lisenet](https://www.lisenet.com/2019/ansible-sample-exam-for-ex294/)

# Task 1:
- for building the ansible.cfg file, you can use the `/etc/ansible/ansible.cfg` file as a reference to get necessary fields to fill
- use ini format for building inventory

# Task 2:
- will use adhoc commands in shell scripts
- do not forget to make the shell executable
- use `authorized_key` module to copy public key from control node to target node. (`copy` module can also be used)
- when using `authorized_key`, the state must be defined as present
- writing to file for privilege escalation configuration can be done using copy or `lineinfile` modules

# Task 3:
- when running commands on different groups of servers two approaches can be taken, first, using `group_vars` and naming files as with its group name. second, using magic variables (`ansible_hostname`, `inventory_hostname`, `groups['all']`)
- use `copy` module as it overwrites the content of a file
- ``lineinfile`` can also be used

# Task 4:
- path of sshd config file is `/etc/ssh/sshd_config`
- use `lineinfile` module with regexp to find the patterns
- take a look the config file to know what the pattern starts with # or the word directly
- use loops since we will use `lineinfile` multiple times
- use handler to restart the sshd service after modifaction

# Task 5: 
- run `ansible-vault create <file_name>`
- set password for vault file
- store password to file `vault_key` as plain text

# Task 6:
- include `user_list.yml` and `secret.yml` file using `vars_files`
- use loop to create users in `webservers` or `database` server
- use magic variable (`ansible_hostname` or `inventory_hostname`, and `groups`) for conditions
- use `include_tasks` inside tasks for code reusability
- use filter to hash password using `| password_hash('sha512')`
- use `lookup('file', 'public_key_path')` with `authorized_key` module to fetch public key from control to managed nodes
- use filtering to convert to string and extract first character for comparison `| string | first`
- to run playbook: `ansible-playbook --vault-password-file vault_key users.yml`

# Task 7:
- use `cron` module
- use `ansible-doc` to see examples of `cron` module

# Task 8: 
- use `ansible-doc`to see examples of `yum_repository` module

# Task 9:
- use `ansible-galaxy init <role_name>` inside the roles directory
- use the `ansible-doc` for the following modules: `parted`, `lvg`, `lvol`, `filesystem`, `mount`
- use `include_tasks`
- use the `ansible-doc` for the following modules: `yum`, `firewalld`, `service`, `pip`, `mysql_user`, and `template`
- to install mysql, use `mysql-server` as the name in `yum` 
- to install pymysql, use `pip` (necessary prereqs for `mysql_user` which sets passwords for database users)
- path to mysql configuration `/etc/my.cnf`
- use `roles` in the `mysql.yml` file to specify the roles used in playbook
*Note:* when setting password in `mysql_user` module do not hash the password

# Task 10:
- use previous notes from task 9
- install packages using `yum`module
- use `ansible-doc firewalld` to see examples for `firewalld` module
- use `immediate` option for `firewalld` module
- use `template` and a handler to restart apache service everytime `index.html` is modified
- use `ansible_facts` to get the ipv4 address of the server

*Note:* to show ansible_facts fields do the following:
- run adhoc cmd: `ansible <host> -m setup` (output will be injected format, meaning will find extra "ansible_" before fields)
- run a playbook using debug and set var: `ansible_facts`

# Task 11: ansible galaxy is not covered in rhce exam

# Task 12:
- using rhel system roles 
- find documentation in `/usr/share/doc/rhel-system-roles/`
- read the `README.md` for examples and what variables to set
- include the system role used either inside a task or inside roles
- role name: `linux-system-roles.selinux`
*Note:* remeber to not overwrite `roles_path` in config file
  
# Task 13: 
- use `setup` module to find the necessary fields "memfree_mb"
- `vm.swapiness` is part of the `sysctl` module use ansible-doc to get more info
- use `fail` module to display error messages

# Task 14: 
- use `lineinfile` module to write to file
- use `archive` module to create an archive of the created file
- use `ansible-doc` to see examples

# Task 15:
- to create custom facts persistently on target machines, create a directory on target machines in `/etc/ansible/facts.d`
- use `blockinfile` combined with `|` to write multiline file
- the created file using `blockinfile` should have `.fact` extension
- assign the `owner` and `group` so that it is accessible by remote user without root privileges

# Task 16: 
- use `vars` to set the packages for proxy and database groups
- use conditions with magic vars to determine which host group
- `ansible_hostname` or `inventory_hostname` in `groups["proxy"]`

*Note:* you can run debug with `var=inventroy_hostname` or `var=hostvar` or `var=ansible_hostname` (if not adhoc cuz must be discovered first)

# Task 17:
- use the `command` module
- use `systemctl set-default <target>`

# Task 18:
- use for loop in jinja2 template iterating all hosts in groups
- use magic variables in the jinja2 template

*Note:* if you are iterating over `groups["all"]` and you want to access `ansible_facts`, then discovery must happen to all servers in inventroy. hence use `hosts: all` and use conditions to target which server the template task should run on


