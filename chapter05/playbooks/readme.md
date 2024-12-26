
## Defining Variables in Playbooks
playbook 中的 vars 可以放到单独的文件里
```yaml
# 以往
vars:
  my_var: my_val
# 替代
vars_files:
  - nginx.yml
```
第四章讲过，和 hosts / group 关联的 vars. 在子文件夹 group_vars 对应在 hosts文件里面定义的 hosts, host_vars 子文件夹下，对应单个 host,比如
```shell
inventory/
  production/
    hosts
    group_vars/
      webservers.yml
      all.yml
    host_vars/
      hostname.yml
```

### 通过 debug 来打印想要看的变量
```yml
- debug: var=myvarname
---

- name: Display the variable
  debug:
    msg: "The file used was {{ conf_file }}"

- name: Concatenate variables
  debug:
    msg: "The URL is https://{{ server_name ~'.'~ domain_name
}}/"
```

## 注册 vars Registering vars
用 register 语法注册 vars，output会是 字典型
```yml
- name: Capture output of whoami command
  command: whoami
  register: login
# example
---
- name: Show return value of command module
  hosts: fedora
  gather_facts: false
  tasks:
    - name: Capture output of id command
      command: id -un
      register: login
    - debug: var=login
    - debug: msg="Logged in as user {{ login.stdout }}"
...
# example how to use
- name: Capture output of id command
  command: id -un
  register: login
- debug: msg="Logged in as user {{ login.stdout }}"

# example ignore task run failure and print the error msg
- name: Run myprog
  command: /opt/myprog
  register: result
  ignore_errors: true
- debug: var=result

# to access keys in a directory ,below are all good
result['stat']['mode']
result['stat'].mode
result.stat['mode']
result.stat.mode
# 甚至这样也行。不用加引号
debug: var=result['stat'][stat_key]
```

## Facts
```yaml
---
- name: 'Ansible facts.'
  hosts: all
  gather_facts: true
  tasks:
  - name: Print out operating system details
    debug:
    msg: >-
      os_family:
      {{ ansible_facts.os_family }},
      distro:
      {{ ansible_facts.distribution }}
      {{ ansible_facts.distribution_version }},
      kernel:
      {{ ansible_facts.kernel }}
...
```

### 查看 server 的所有 Facts
用 setup module
```shell
ansible ubuntu -m setup
# 结果是一个 key 为 ansible_facts 的 map

# 在ansible中如果一个模块返回字典，并包含 ansible_facts 在key中，ansible会为那些key 在 environment中创建variable names,并把他们关联到host上。Modules that return information about objects that are not unique for the host have their name ending in _info.

# 那些 return facts的module不用使用 register。因为 ansible 自动创建，比如 service_facts 模块

# 或者用filter 过滤
$ ansible all -m setup -a filter=ansible_all_ipv6_addresses'
```

### Local Facts
还有办法可以把 facts associate 到 某个 hosts上
为 remote host machine 在 /etc/ansible/facts.d 文件夹下创建如下类型的文件
.ini json 可执行文件(不接参数，输出json io)

这些 facts 存放在 ansible_local的 特殊 variable 中

比如把如下文件放到 remote host中
```ini /etc/ansible/facts.d/example.fact
[book]
title=Ansible: Up and Running
authors=Meijer, Hochstein, Moser
publisher=O'Reilly
```
使用：
```yml
- name: Print ansible_local
  debug: var=ansible_local

- name: Print book title
  debug: msg="The title of the book is {{ansible_local.example.book.title }}"
```

### 用 set_fact 来定义一个新的 var

set_fact 是一个 ansible 的 module
比如在 service_facts 被取得之后马上用 set_fact来定义一个从他里面取得的var

```yaml
- name: Set nginx_state
  when: ansible_facts.services.nginx.state is defined
  set_fact:
    nginx_state: "{{ ansible_facts.services.nginx.state }}"
```

## built-in vars

## 在 command line 设置 vars
**-e var=value** 它的优先级最高

## 优先级排列
详情见书中