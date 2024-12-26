
## inventory & plugin

inventory can be : a file(in many fomats),a directory, or an executable, and some executables are bundled as plug-ins

an ansible.cfg which enables all inventory plug-ins 
```conf
[defaults]
inventory = inventory
[inventory]
enable_plugins = host_list, script, auto, yaml, ini, toml
```
**inventory plug-ins** allow us to point at data sources, like your cloud provider, to compile the inventory

An inventory can be stored separately from your playbooks,这样你的 playbook 可以针对不同的目标去执行，aws vm azure

## inventory/ hosts files

ansible 中默认注册 host 的形式是把他们列在一个 txt file 中, 这个 file 叫做 **inventory hosts files**

最简单的形式：txt 文件 hosts
```host
frankfurt.example.com
helsinki.example.com
hongkong.example.com
johannesburg.example.com
london.example.com
newyork.example.com
seoul.example.com
sydney.example.com
```

ansible add *localhost* by default ,ansible 会直接跟他交互，而不是用 ssh

## launch 多个 vagrant 虚拟机，实验用

### 为三台名字相似的主机配置 ssh config
3个虚拟机启动之后，他们的配置只有端口有所不同，而 ansible 用本机的 ssh 来连接远程，这意味着，在 SSH config file 中设置的任何主机别名他都认识。因此需要编辑 ~/.ssh/config 文件，在里面配置适用于3 台 vagrant 主机的配置：
```ini
Host vagrant*
Hostname 127.0.0.1
User vagrant
UserKnownHostsFile /dev/null
StrictHostKeyChecking no
PasswordAuthentication no
IdentityFile ~/.vagrant.d/insecure_private_key/vagrant.key.rsa
IdentitiesOnly yes
LogLevel FATAL
```
vagrant* 指代 vagrant1 vagrant2 vagrant3 

### 为3 台 vagrant 主机配置 inventory/host 文件
```
vagrant1 ansible_port=2222
vagrant2 ansible_port=2200
vagrant3 ansible_port=2201
```

### 测试连接

```bash
ansible vagrant2 -a "ip addr show dev enp0s3"
```

## behavioral inventory parameters 

上面的 **ansible_port** 用来指定通过哪个端口连接agrant，这种参数叫做 `behavioral inventory parameters`,这样的参数还有很多

**ansible_connection**
- ansible 传输方式，默认用 python 的 `paramiko`，设置为 smart后，会检查 本地的 SSH client 是否支持 ControlPersist, 支持的话 ansible 就会用本地的 SSH client，fouze `paramiko`

**ansible_shell_type**

详细列表见书中
```
ansible_host 
ansible_port 
ansible_user 
ansible_password
ansible_connection
ansible_ssh_private_key_file
ansible_shell_type
ansible_python_interpreter
ansible_*_interpreter
```

### Changing Behavioral Parameter Defaults
在 inventory file 可以覆盖 Behavioral parameter,也可以在 ansible.cfg 文件的 defaults 那块改，在哪儿改看你的需求。

**记住可以在 ~/.ssh/config 改ssh相关的 preference**

ansible.cfg 的 Behavioral inventory parameter 参考书中
```
ansible_port
ansible_user
ansible_ssh_private_key_file
ansible_shell_type
```
>ansible.cfg 的 ansible_shell_type 和 inventory 中的不一样，ansible.cfg 里面指定绝对路径 inventory 里面会查这个路径的 basenamne (/usr/local/bin/fish -> fish),用的是 fish

## group group group 

我们一般是操作1组 hossts，而不是单个。ansible 默认定义一个分组叫 all/*，他代表了 invento 所有的 hosts 比如 我们可以用这个命令 查看 所有及其上的时间是不是都一致

```shell
ansible all -a "date"
# or
ansible '*' -a "date"
```

定义可分组的 inventory 我们用 **.ini** 文件

```ini

newyork.example.com
seoul.example.com
sydney.example.com
[vagrant]
vagrant1 ansible_port=2222
vagrant2 ansible_port=2200
vagrant3 ansible_port=2201
```

一个 host 可以在 group 之外，同时也可以在 group 之内再定义一次

vagrant1 2 3 是 aliases，ansible 通过以下方式解析 hostnames：inventory，ssh config(~/.ssh/config)，/etc/hosts 以及 DNS.ansible 当然也识别 <ip>:port - 127.0.0.1:2222 这种形式,但是我们现在操作三个vagrant虚拟机的例子里我们如果这样写就不形，因为 Ansible’s inventory can associate only a single host with 127.0.0.1
```
[vagrant]
127.0.0.1:2222
127.0.0.1:2200
127.0.0.1:2201
```

### Groups of Groups
可以定义包含别的 group 的 group
goup of groups 里面不能写的是host,必须是group
`children`好像是固定写法
```
[django:children]
web
task
```

### Numbered Hosts
```ini
[web]
web[1:20].example.com
```
代表 web1.example.com web2.example.com

**web[01:20].example.com** 这样也行

```ini
[web]
web-[a:t].example.com
```
代表 **web-a.example.com, web-b.example.com**

## Hosts & Group Variables: Inside the Inventory

之前我们在 inventory中定义 behavioral inventory parameters
比如下面的 ansible_host / ansible_port

```ini
vagrant1 ansible_host=127.0.0.1 ansible_port=2222
```
我们也可以定义任意的 behaive parameter,比如这里的 red
```ini
amsterdam.example.com color=red
```
我们一般定义 group level 的 parameter ，比如按环境来分组

```ini
; 格式为[<groupname>:vars]
; all group 给全局用
[all:vars]
ntp_server=ntp.ubuntu.com

[production:vars]
db_primary_host=frankfurt.example.com
db_primary_port=5432

[staging:vars]
db_primary_host=chicago.example.com
db_primary_port=5432

[vagrant:vars]
db_primary_host=vagrant3
db_primary_port=5432
```



## host & group vars in their own files
如果服务器多，inventory会变得很长
并且在 inventory file 中只能用 bool 和 string

可以给 每一个 host & group 配置各自单独的 host and group variables
ansible 规定
host variable files 在 host_vars 目录下 
group variable files 在 group_vars 目录下
可以在 playbook 目录下(优先)，也可以在 inventory旁边的目录下

for example
```shell
/home/lorin/playbooks/
/home/lorin/inventory/hosts

/home/lorin/inventory/host_vars/amsterdam.example.com （file）
/home/lorin/inventory/group_vars/production （file）
```

或者也可以在group_vars放一个 production文件夹，然后创建相关的 group var 文件，比如 
```shell
group_vars/production/db
group_vars/production/rabbitmq
```

## dynamic inventory
aws azure 这种云服务可以通过cli / api获取机器列表
所以可以写一个 动态 inverntory 来动态获取
把 inventory 文件标记为可执行，ansible就会执行这个 inventroy而不是读取
比如 chomd +x vagrant.py

### inventory plug-in
查看插件
To see the list of available plug-ins:
**$ ansible-doc -t inventory -l**
To see plug-in-specific documentation and examples:
**$ ansible-doc -t inventory <plugin name>**

- ec2 插件
现安装 pip3 install boto3 botocore
然后创建 inventory/aws_ec2.yml

最后一行写上
plugin: aws_ec2

- Azure 插件
pip3 install msrest msrestazure

inventory/azure_rm.yml
```yaml
plugin: azure_rm
platform: azure_rm
auth_source: auto
plain_host_names: true
```

### Dynamic Inventory Script
需要提供两个支持 --host --list
返回 ansible 所能识别的json 

host是获取单个信息，就像
ansible-inventory -i inventory/hosts --host=vagrant2

list返回是这样
```json
{"vagrant": ["vagrant1", "vagrant2", "vagrant3"]}
```
还可以帮ansible返回 behevial parameters

```json
"_meta": {
"hostvars": {
"vagrant1": {
"ansible_user": "vagrant",
"ansible_host": "127.0.0.1",
"ansible_ssh_private_key_file":
"~/.vagrant.d/insecure_private_key",
"ansible_port": "2222"
},
"vagrant2": {
"ansible_user": "vagrant",
"ansible_host": "127.0.0.1",
"ansible_ssh_private_key_file":
"~/.vagrant.d/insecure_private_key",
"ansible_port": "2200"
},
```

### 写动态脚本
用vagrant举例
vagrant status
vagrant status --machine-readable
vagrant ssh-config vagrant2
要在python脚本中利用这几个 vagrant 给的方法

## 把 Inventory 弄成多个文件
如果有多个静态/动态 inventory 文件，把它们放在同一目录下，然后通过如下方法设置ansible 使用该个目录

- ansible.cfg 的 inventory parameter 参数
- 在命令行中使用 -i flag

ansible 就会把所有文件处理成同一个 inventory

## 通过 add_host 和 group_by 在 ansible 运行时添加 entry
### add_host
add_host 添加 host 到 inventory, 如果你是申请了新的 vm 这很有用
```yaml
---
- name: Add the host
  add_host
    name: hostname
    groups: web,staging
    myvar: myval
```

`creates: Vagrantfile` 保持幂等性,如果有了就不做操作

```yml
- name: Create a Vagrantfile
  command: "vagrant init {{ box }}"
  args:
    creates: Vagrantfile
```

### group_by
group_by 允许你在 playbook 运行时创建新的 group.
比如如果启用了 ansible 收集 fact , ansible 会关联很多 facts 到 host上面，ansible_machine就是一个其中一个 fact
指示host 的 cpu 架构，我们就可以把 vm 按 cpu 架构来分组
```yml
---
- name: Group hosts by distribution
  hosts: all
  gather_facts: true
  tasks:
  - name: Create groups based on distro
    group_by:
      key: "{{ ansible_facts.distribution }}"
- name: Do something to Ubuntu hosts
  hosts: Ubuntu
  become: true
  tasks:
    - name: Install jdk and jre
    apt:
      update_cache: true
      name:
      - openjdk-11-jdk-headless
      - openjdk-11-jre-headless
- name: Do something else to CentOS hosts
  hosts: CentOS
  become: true
  tasks:
    - name: Install jdk
      yum:
        name:
          - java-11-openjdk-headless
          - java-11-openjdk-devel
```