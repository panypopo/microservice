#Ansible 快速上手
##Ansible简介
ansible是新出现的自动化运维工具，基于Python开发，集合了众多运维工具（puppet、cfengine、chef、func、fabric）的优点，实现了批量系统配置、批量程序部署、批量运行命令等功能。ansible是基于模块工作的，本身没有批量部署的能力。真正具有批量部署的是ansible所运行的模块，ansible只是提供一种框架。主要包括：

* 连接插件connection plugins：负责和被监控端实现通信；
* host inventory：指定操作的主机，是一个配置文件里面定义监控的主机；
* 各种模块核心模块、command模块、自定义模块；
* 借助于插件完成记录日志邮件等功能；
* playbook：可以让节点一次性运行多个任务。

###架构图
![ansible](http://s3.51cto.com/wyfs02/M02/53/A7/wKiom1Rsxz3ToUCAAAGROYAM3EI989.jpg)

###特性

* no agents：不需要在被管控主机上安装任何客户端；
* no server：无服务器端，使用时直接运行命令即可；
* modules in any languages：基于模块工作，可使用任意语言开发模块；
* yaml，not code：使用yaml语言定制剧本playbook；
* ssh by default：基于SSH工作；
* strong multi-tier solution：可实现多级指挥。

###工作流程
![Ansible工作流程](http://s3.51cto.com/wyfs02/M01/53/A7/wKiom1Rsx2uQYJZ5AAJplY08vOQ976.jpg)

##Ansible安装配置
###安装Ansible
Ansible 能够安装到 Linux、BSD、Mac OS X 等平台，Python 版本最低要求为 2.6。常用 Linux 发行一般可以通过其自带的包管理器[安装 Ansible](http://www.ansibleworks.com/docs/intro_installation.html)：

~~~
yum install ansible     # RHEL/CentOS/Fedora，需要配置 EPEL
apt-get install ansible # Debian/Ubuntu
emerge -avt ansible     # Gentoo/Funtoo
~~~

如果你在所用 Linux 发行版的包仓库中找不到 Ansible，那么也可以通过 pip 来安装 Ansible，同时也会安装 paramiko、PyYAML、jinja2 等 Python 依赖库。

~~~
pip install ansible
~~~

本例采用ubuntu14.04:

~~~
root@b8f58c772bf9:/# apt-get install ansible
~~~

安装后验证：

~~~
root@b8f58c772bf9:/# ansible --version
ansible 1.5.4
~~~

###受控机安装SSH Server

~~~
> apt-get install openssh-server
> service ssh start
~~~

###配置免密码登陆

####生成公钥和密钥
在管理机器上执行以下语句，后续一直按回车即可

~~~sh
ssh-keygen -t rsa
~~~

会在~/.ssh/下生成id_rsa  id_rsa.pub这两个文件。

~~~sh
cp id_rsa.pub authorized_keys
~~~

将authorized_keys拷贝一份到受控机上（运行以下语句）

~~~sh
scp authorized_keys pany@172.17.0.4:/home/pany/.ssh
~~~

登陆受控机运行以下命令：

~~~sh
chmod 600 ~/.ssh/authorized_keys 
~~~

在管理器上使用SSH登陆（第一次需要密码，以后不再需要密码）

~~~sh
ssh 172.17.0.4 -l pany
~~~

###准备 Inventory
Inventory 文件用来定义你要管理的主机。其默认位置在 /etc/ansible/hosts ，如果不保存在默认位置，也可通过 -i 选项指定。被管理的机器可以通过其 IP 或域名指定。未分组的机器需保留在 hosts 的顶部，分组可以使用 [] 指定，如：

~~~
[web]  
linuxtoy.org
~~~

同时，分组也能嵌套

~~~
[vps:children]  
web  
db
~~~

此外，也可以通过数字和字母模式来指定一系列连续主机，如：

~~~
[1:3].linuxtoy.org # 等价于
1.linuxtoy.org、2.linuxtoy.org、3.linuxtoy.org  
[a:c].linuxtoy.org # 等价于
a.linuxtoy.org、b.linuxtoy.org、c.linuxtoy.org
~~~

###Ansible配置文件
配置文件可以从多个地方加载，其优先级顺序为：

* ANSIBLE_CONFIG (环境变量)
* ansible.cfg (当前目录)
* .ansible.cfg (home目录)
* /etc/ansible/ansible.cfg

示例配置：

~~~
# config file for ansible -- http://ansible.com/
# ==============================================

# nearly all parameters can be overridden in ansible-playbook
# or with command line flags. ansible will read ANSIBLE_CONFIG,
# ansible.cfg in the current working directory, .ansible.cfg in
# the home directory or /etc/ansible/ansible.cfg, whichever it
# finds first

[defaults]

# some basic default values...

#inventory      = /etc/ansible/hosts
#library        = /usr/share/my_modules/
#remote_tmp     = $HOME/.ansible/tmp
#local_tmp      = $HOME/.ansible/tmp
#forks          = 5
#poll_interval  = 15
#sudo_user      = root
#ask_sudo_pass = True
#ask_pass      = True
#transport      = smart
#remote_port    = 22
#module_lang    = C
#module_set_locale = False

# plays will gather facts by default, which contain information about
# the remote system.
#
# smart - gather by default, but don't regather if already gathered
# implicit - gather by default, turn off with gather_facts: False
# explicit - do not gather by default, must say gather_facts: True
#gathering = implicit

# by default retrieve all facts subsets
# all - gather all subsets
# network - gather min and network facts
# hardware - gather hardware facts (longest facts to retrieve)
# virtual - gather min and virtual facts
# facter - import facts from facter
# ohai - import facts from ohai
# You can combine them using comma (ex: network,virtual)
# You can negate them using ! (ex: !hardware,!facter,!ohai)
# A minimal set of facts is always gathered.
#gather_subset = all

# some hardware related facts are collected
# with a maximum timeout of 10 seconds. This
# option lets you increase or decrease that
# timeout to something more suitable for the
# environment. 
# gather_timeout = 10

# additional paths to search for roles in, colon separated
#roles_path    = /etc/ansible/roles

# uncomment this to disable SSH key host checking
#host_key_checking = False

# change the default callback
#stdout_callback = skippy
# enable additional callbacks
#callback_whitelist = timer, mail

# Determine whether includes in tasks and handlers are "static" by
# default. As of 2.0, includes are dynamic by default. Setting these
# values to True will make includes behave more like they did in the
# 1.x versions.
#task_includes_static = True
#handler_includes_static = True

# change this for alternative sudo implementations
#sudo_exe = sudo

# What flags to pass to sudo
# WARNING: leaving out the defaults might create unexpected behaviours
#sudo_flags = -H -S -n

# SSH timeout
#timeout = 10

# default user to use for playbooks if user is not specified
# (/usr/bin/ansible will use current user as default)
#remote_user = root

# logging is off by default unless this path is defined
# if so defined, consider logrotate
#log_path = /var/log/ansible.log

# default module name for /usr/bin/ansible
#module_name = command

# use this shell for commands executed under sudo
# you may need to change this to bin/bash in rare instances
# if sudo is constrained
#executable = /bin/sh

# if inventory variables overlap, does the higher precedence one win
# or are hash values merged together?  The default is 'replace' but
# this can also be set to 'merge'.
#hash_behaviour = replace

# by default, variables from roles will be visible in the global variable
# scope. To prevent this, the following option can be enabled, and only
# tasks and handlers within the role will see the variables there
#private_role_vars = yes

# list any Jinja2 extensions to enable here:
#jinja2_extensions = jinja2.ext.do,jinja2.ext.i18n

# if set, always use this private key file for authentication, same as
# if passing --private-key to ansible or ansible-playbook
#private_key_file = /path/to/file

# If set, configures the path to the Vault password file as an alternative to
# specifying --vault-password-file on the command line.
#vault_password_file = /path/to/vault_password_file

# format of string {{ ansible_managed }} available within Jinja2
# templates indicates to users editing templates files will be replaced.
# replacing {file}, {host} and {uid} and strftime codes with proper values.
#ansible_managed = Ansible managed: {file} modified on %Y-%m-%d %H:%M:%S by {uid} on {host}
# This short version is better used in templates as it won't flag the file as changed every run.
#ansible_managed = Ansible managed: {file} on {host}

# by default, ansible-playbook will display "Skipping [host]" if it determines a task
# should not be run on a host.  Set this to "False" if you don't want to see these "Skipping"
# messages. NOTE: the task header will still be shown regardless of whether or not the
# task is skipped.
#display_skipped_hosts = True

# by default, if a task in a playbook does not include a name: field then
# ansible-playbook will construct a header that includes the task's action but
# not the task's args.  This is a security feature because ansible cannot know
# if the *module* considers an argument to be no_log at the time that the
# header is printed.  If your environment doesn't have a problem securing
# stdout from ansible-playbook (or you have manually specified no_log in your
# playbook on all of the tasks where you have secret information) then you can
# safely set this to True to get more informative messages.
#display_args_to_stdout = False

# by default (as of 1.3), Ansible will raise errors when attempting to dereference
# Jinja2 variables that are not set in templates or action lines. Uncomment this line
# to revert the behavior to pre-1.3.
#error_on_undefined_vars = False

# by default (as of 1.6), Ansible may display warnings based on the configuration of the
# system running ansible itself. This may include warnings about 3rd party packages or
# other conditions that should be resolved if possible.
# to disable these warnings, set the following value to False:
#system_warnings = True

# by default (as of 1.4), Ansible may display deprecation warnings for language
# features that should no longer be used and will be removed in future versions.
# to disable these warnings, set the following value to False:
#deprecation_warnings = True

# (as of 1.8), Ansible can optionally warn when usage of the shell and
# command module appear to be simplified by using a default Ansible module
# instead.  These warnings can be silenced by adjusting the following
# setting or adding warn=yes or warn=no to the end of the command line
# parameter string.  This will for example suggest using the git module
# instead of shelling out to the git command.
# command_warnings = False


# set plugin path directories here, separate with colons
#action_plugins     = /usr/share/ansible/plugins/action
#cache_plugins      = /usr/share/ansible/plugins/cache
#callback_plugins   = /usr/share/ansible/plugins/callback
#connection_plugins = /usr/share/ansible/plugins/connection
#lookup_plugins     = /usr/share/ansible/plugins/lookup
#inventory_plugins  = /usr/share/ansible/plugins/inventory
#vars_plugins       = /usr/share/ansible/plugins/vars
#filter_plugins     = /usr/share/ansible/plugins/filter
#test_plugins       = /usr/share/ansible/plugins/test
#strategy_plugins   = /usr/share/ansible/plugins/strategy

# by default callbacks are not loaded for /bin/ansible, enable this if you
# want, for example, a notification or logging callback to also apply to
# /bin/ansible runs
#bin_ansible_callbacks = False


# don't like cows?  that's unfortunate.
# set to 1 if you don't want cowsay support or export ANSIBLE_NOCOWS=1
#nocows = 1

# set which cowsay stencil you'd like to use by default. When set to 'random',
# a random stencil will be selected for each task. The selection will be filtered
# against the `cow_whitelist` option below.
#cow_selection = default
#cow_selection = random

# when using the 'random' option for cowsay, stencils will be restricted to this list.
# it should be formatted as a comma-separated list with no spaces between names.
# NOTE: line continuations here are for formatting purposes only, as the INI parser
#       in python does not support them.
#cow_whitelist=bud-frogs,bunny,cheese,daemon,default,dragon,elephant-in-snake,elephant,eyes,\
#              hellokitty,kitty,luke-koala,meow,milk,moofasa,moose,ren,sheep,small,stegosaurus,\
#              stimpy,supermilker,three-eyes,turkey,turtle,tux,udder,vader-koala,vader,www

# don't like colors either?
# set to 1 if you don't want colors, or export ANSIBLE_NOCOLOR=1
#nocolor = 1

# if set to a persistent type (not 'memory', for example 'redis') fact values
# from previous runs in Ansible will be stored.  This may be useful when
# wanting to use, for example, IP information from one group of servers
# without having to talk to them in the same playbook run to get their
# current IP information.
#fact_caching = memory


# retry files
# When a playbook fails by default a .retry file will be created in ~/
# You can disable this feature by setting retry_files_enabled to False
# and you can change the location of the files by setting retry_files_save_path

#retry_files_enabled = False
#retry_files_save_path = ~/.ansible-retry

# squash actions
# Ansible can optimise actions that call modules with list parameters
# when looping. Instead of calling the module once per with_ item, the
# module is called once with all items at once. Currently this only works
# under limited circumstances, and only with parameters named 'name'.
#squash_actions = apk,apt,dnf,package,pacman,pkgng,yum,zypper

# prevents logging of task data, off by default
#no_log = False

# prevents logging of tasks, but only on the targets, data is still logged on the master/controller
#no_target_syslog = False

# controls whether Ansible will raise an error or warning if a task has no
# choice but to create world readable temporary files to execute a module on
# the remote machine.  This option is False by default for security.  Users may
# turn this on to have behaviour more like Ansible prior to 2.1.x.  See
# https://docs.ansible.com/ansible/become.html#becoming-an-unprivileged-user
# for more secure ways to fix this than enabling this option.
#allow_world_readable_tmpfiles = False

# controls the compression level of variables sent to
# worker processes. At the default of 0, no compression
# is used. This value must be an integer from 0 to 9.
#var_compression_level = 9

# controls what compression method is used for new-style ansible modules when
# they are sent to the remote system.  The compression types depend on having
# support compiled into both the controller's python and the client's python.
# The names should match with the python Zipfile compression types:
# * ZIP_STORED (no compression. available everywhere)
# * ZIP_DEFLATED (uses zlib, the default)
# These values may be set per host via the ansible_module_compression inventory
# variable
#module_compression = 'ZIP_DEFLATED'

# This controls the cutoff point (in bytes) on --diff for files
# set to 0 for unlimited (RAM may suffer!).
#max_diff_size = 1048576

[privilege_escalation]
#become=True
#become_method=sudo
#become_user=root
#become_ask_pass=False

[paramiko_connection]

# uncomment this line to cause the paramiko connection plugin to not record new host
# keys encountered.  Increases performance on new host additions.  Setting works independently of the
# host key checking setting above.
#record_host_keys=False

# by default, Ansible requests a pseudo-terminal for commands executed under sudo. Uncomment this
# line to disable this behaviour.
#pty=False

[ssh_connection]

# ssh arguments to use
# Leaving off ControlPersist will result in poor performance, so use
# paramiko on older platforms rather than removing it, -C controls compression use
#ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s

# The path to use for the ControlPath sockets. This defaults to
# "%(directory)s/ansible-ssh-%%h-%%p-%%r", however on some systems with
# very long hostnames or very long path names (caused by long user names or
# deeply nested home directories) this can exceed the character limit on
# file socket names (108 characters for most platforms). In that case, you
# may wish to shorten the string below.
#
# Example:
# control_path = %(directory)s/%%h-%%r
#control_path = %(directory)s/ansible-ssh-%%h-%%p-%%r

# Enabling pipelining reduces the number of SSH operations required to
# execute a module on the remote server. This can result in a significant
# performance improvement when enabled, however when using "sudo:" you must
# first disable 'requiretty' in /etc/sudoers
#
# By default, this option is disabled to preserve compatibility with
# sudoers configurations that have requiretty (the default on many distros).
#
#pipelining = False

# if True, make ansible use scp if the connection type is ssh
# (default is sftp)
#scp_if_ssh = True

# if False, sftp will not use batch mode to transfer files. This may cause some
# types of file transfer failures impossible to catch however, and should
# only be disabled if your sftp version has problems with batch mode
#sftp_batch_mode = False

[accelerate]
#accelerate_port = 5099
#accelerate_timeout = 30
#accelerate_connect_timeout = 5.0

# The daemon timeout is measured in minutes. This time is measured
# from the last activity to the accelerate daemon.
#accelerate_daemon_timeout = 30

# If set to yes, accelerate_multi_key will allow multiple
# private keys to be uploaded to it, though each user must
# have access to the system via SSH to add a new key. The default
# is "no".
#accelerate_multi_key = yes

[selinux]
# file systems that require special treatment when dealing with security context
# the default behaviour that copies the existing context or uses the user default
# needs to be changed to use the file system dependent context.
#special_context_filesystems=nfs,vboxsf,fuse,ramfs

# Set this to yes to allow libvirt_lxc connections to work without SELinux.
#libvirt_lxc_noseclabel = yes

[colors]
#highlight = white
#verbose = blue
#warn = bright purple
#error = red
#debug = dark gray
#deprecate = purple
#skip = cyan
#unreachable = red
#ok = green
#changed = yellow
#diff_add = green
#diff_remove = red
#diff_lines = cyan
~~~


##Ansible使用
###Hello World

~~~sh
root@b8f58c772bf9:~#  ansible all -a "/bin/echo hello" -u pany
172.17.0.4 | success | rc=0 >>
hello
~~~

###执行Ad-Hoc命令
使用-a指定命令， -f指定并发数（默认为5）, -m是选择使用的模块, -u指定用户

####执行shell

~~~sh
# 打印hello
ansible all -a "/bin/echo hello"

# 重启
ansible all -a "/sbin/reboot" -f 10 --sudo -K

# 使用shell模块 注意当前shell的引号问题
ansible all -m shell -a 'ls -al ~'
~~~

####文件操作

~~~sh
# 传输文件
ansible all -m copy -a "src=~/projects/tests/t.py dest=~"

# 修改文件权限，所有者，分组（这些参数可以用在copy模块中）
ansible all -m file -a "dest=~/t.py mode=777 owner=ashin group=ashin"

# 创建文件夹
ansible all -m file -a "dest=~/tests mode=755 owner=ashin group=ashin state=directory"

# 删除文件夹
ansible all -m file -a "dest=~/tests state=absent"
~~~

####安装软件
使用yum或apt模块可以进行软件的安装

~~~sh
# 确保某个程序已经安装，并保持当前版本，如果没安装则进行安装
ansible v1 -m apt -a "name=python-pip state=present" --sudo -K

# 确保安装最新版本
ansible v1 -m apt -a "name=git state=latest"

# 确保没有安装某个程序，安装了则卸载
ansible v1 -m apt -a "name=git state=absent" --sudo -K
~~~

####用户管理

~~~sh
# 新建用户
ansible all -m user -a "name=foo password=foo" --sudo -K

# 删除用户
ansible all -m user -a "name=foo state=absent" --sudo -K
~~~

####Git

~~~sh
# 使用https方式检出代码到本地，前提是要先通过apt模块安装git
# dest目录必须是空文件夹或者还不存在的文件夹

ansible v1 -m git -a "repo=https://github.com/axiaoxin/json_line_format.vim.git dest=~/project-dir version=HEAD"
~~~

####系统服务管理

~~~sh
# 确保某服务已开启，没开则开
ansible v1 -m service -a "name=mysql state=started" --sudo -K
ansible v1 -m service -a "name=mysql state=restarted" --sudo -K
ansible v1 -m service -a "name=mysql state=stopped" --sudo -K
~~~

####超时和后台操作

~~~sh
#3600秒timeout -P 0 表示不需要轮询
#此命令会返回一个job id
ansible all -B 3600 -P 0 -a "/usr/bin/long_running_operation --do-stuff"

#查询job状态
ansible web1.example.com -m async_status -a "jid=488359678239.2844"

#30分钟timeout，每60秒检查一次状态
ansible all -B 1800 -P 60 -a "/usr/bin/long_running_operation --do-stuff"
~~~

***

###Playbooks
Playbooks使用yaml语法，在ansible中几乎所有的yaml文件都是以list开始，每个item是键值对的list。

所有的yaml文件都以---开头表示开始一个document，所有的列表元素以-开头，键值对用:，后面的空格是必须的。键值对中的值如果是bool类型的字符串true/false（首字母不论大小写），pyyaml会load成python中对应的bool值，在键值对中如果值中有:或者以{{开头的变量定义，则必须用引号引起来.

示例 test.yml：

~~~yaml
---
- hosts: v1
remote_user: ashin
tasks:
    - name: test connection
    ping:
    remote_user: ashin
~~~

运行：

~~~sh
ansible-playbook test.yml
~~~

####基础
#####Hosts and Users

`hosts` 一个或多个host，也可以是host通配符，多个host通过`:`分隔。
`remote_user` 远程登录的用户名。

~~~yaml
---
- hosts: webservers
  remote_user: root
~~~

也可以为每个任务定义`remote_user`：

~~~yaml
---
- hosts: webservers
  remote_user: root
  tasks:
    - name: test connection
      ping:
      remote_user: yourname
~~~

支持变身too ：

~~~yaml
---
- hosts: webservers
  remote_user: yourname
  become: yes
  become_user: postgres
~~~

sudo:

~~~yaml
---
- hosts: webservers
  remote_user: yourname
  tasks:
    - service: name=nginx state=started
      become: yes
      become_method: sudo
~~~

su:

~~~yaml
---
- hosts: webservers
  remote_user: yourname
  become: yes
  become_method: su
~~~

如果需要为sudo输入密码，则在运行ansible-playbook时加入参数：

~~~sh
ansible-playbook test.yml --ask-become-pass 
ansible-playbook test.yml --K
~~~

#####任务列表
`task` 每一个playbook脚本包含一个任务列表，它将按顺序，一个一个地在对应的host上面执行。
任务的运行是自上而下的，如果有host运行失败，则将被移除host列表。

`service`:

~~~yaml
tasks:
  - name: make sure apache is running
    service: name=httpd state=started
~~~

`command` and `shell` :

~~~yaml
tasks:
  - name: disable selinux
    command: /sbin/setenforce 0
~~~

返回值处理：

~~~yaml
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand || /bin/true
~~~

~~~yaml
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand
    ignore_errors: True
~~~

参数：

使用`{{` `}}`括起来

~~~yaml

vars: 
 - vhost: test
tasks:
  - name: create a virtual host file for {{ vhost }}
    template: src=somefile.j2 dest=/etc/httpd/conf.d/{{ vhost }}
~~~

#####Handlers

~~~yaml
- name: template configuration file
  template: src=template.j2 dest=/etc/foo.conf
  notify:
     - restart memcached
     - restart apache
~~~

~~~yaml
handlers:
    - name: restart memcached
      service: name=memcached state=restarted
      listen: "restart web services"
    - name: restart apache
      service: name=apache state=restarted
      listen: "restart web services"

tasks:
    - name: restart everything
      command: echo "this task will restart the web services"
      notify: "restart web services"
~~~

详细请参照：[Playbook介绍](http://docs.ansible.com/ansible/playbooks_intro.html)



***

####Roles和Include
####include和重用

include和传参

~~~yaml
tasks:
  - include: wordpress.yml wp_user=timmy
  - include: wordpress.yml wp_user=alice
  - include: wordpress.yml wp_user=bob
~~~

或者：

~~~yaml
tasks:
  - include: wordpress.yml
    vars:
        wp_user: timmy
        ssh_keys:
          - keys/one.txt
          - keys/two.txt
~~~

然后在 wordpress.yml 使用 `{{wp_user}}` 引用参数。

####Roles
使用roles可以更好的组织框架，简单例子：

当前目录结构：

~~~
.
├── hosts
├── roles
│   └── common
│       └── tasks
│           └── main.yml
└── site.yml
~~~

site.yml文件是入口，内容为：

~~~yaml
---
- hosts: all
  roles:
    - role: common

- hosts: v1
  tasks:
    - include: roles/common/tasks/main.yml
~~~

commom角色是用于在全部主机上执行的任务，任务为ping，tasks中文件名必须为`main`，其内容为：

~~~yaml
---
- name: test connection
  ping:
~~~

v1角色是通过include直接指定task。

运行：

~~~yaml
ansible-playbook site.yml
~~~

role下有很多结构，ansible会自动按照文件结构进行加载解析。具体目录结构如下：

~~~
.
├── defaults
├── files
├── handlers
├── meta
├── tasks
├── templates
└── vars
~~~

- 如果roles/x/tasks/main.yml存在,则自动将里面的tasks添加到play中。

- 如果roles/x/handlers/main.yml存在,则自动将里面的handlers添加到play中。

- 如果roles/x/vars/main.yml存在, 则自动将其中的variables添加到play中。

- 如果roles/x/meta/main.yml存在,则添加role的依赖关系roles中。

- 任何copy任务、script任务都可以引用roles/x/files中的文件，无论是使用绝对或相对路径都可以。

- 任何template任务都可以引用roles/x/templates中的文件，无论绝对或相对路径。

- 任何include任务都可以引用roles/x/tasks/中的文件，无论相对或绝对路径

具体可以参见[文档](http://docs.ansible.com/ansible/playbooks_roles.html).