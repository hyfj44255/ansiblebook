
## ansible ad-hoc command

建立一个 inventory 文件 xx.ini

```shell
$ ansible testserver -i inventory/vagrant.ini -m ping
```
## 用 ansible.cfg 文件简化

刚才的命令我们需要输入很多参数，然而 ansible 提供了另外一种机制 ansible.cfg 来设定一些默认值

cfg 文件可以指定 inventory file在什么地方，以及 output 格式的一些设置

在 inventory用 var 块来指定 ssh private key ,而不是 ansible.cfg ,因为ansible.cfg 和 inventory有关. ssh private key file 也可以和 ansible.cfg放一起，的那时不灵活。或者依赖于你的SSH 设置

书中的例子在 ansible.cfg disable 了 SSH host-key checking.这样方便，不用每次创建销毁 agrant机器 都更改 ~/.ssh/known_hosts 但是不安全

### ansible.cfg 应该放在哪儿
- env **ANSIBLE_CONFIG**
- ./ansible.cfg 即当前目录
- ~/.ansible.cfg home目录中
- /etc/ansible/ansible.cfg (linux) or /usr/local/etc/ansible/ansible.cfg(*BSD)
一般都放在当期目录下，和playbook 一起。这样方便上传到git，这样叫做 project-based config file

## 命令简化以后
```shell
ansible testserver -m ping
```

**m**: module

**-a**： module 要执行的命令

```shell
ansible testserver -m command -a uptime
$ ansible testserver -a "tail /var/log/dmesg"
```
**-m command**： command 这个 module 用得台普遍了，所以可以省略

**-b or --become**： 切换 root user