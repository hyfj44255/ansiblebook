# 第二章 playbook

上一章不用 playbook，用 add hoc 的方式执行 ansible
```shell
ansible webservers -b -m package -a 'name=nginx update_cache=true'
```
这样 要执行的脚本是一串字符串，容易隐藏bug

## playbook中scr的两个默认路径

``An Ansible convention is to copy files from a subdirectory named files, and
to source Jinja2 templates from a subdirectory named templates. Ansible
searches these directories automatically. We follow this convention
throughout the book.``

在 playbook 中使用 *copy* 模块复制文件模块时，source 默认是 playbook 目录下的 **files/** 文件夹
使用 *template* 模块 默认是 playbook 的 **templates/** 文件夹下
ansible 会自动搜索这些目录

## ansible-inventory --graph
方便查看 inventory图

## Run playbook 

```shell
ansible-playbook webservers.yml
```

webservers.yml 是一个 playbook

## playbook 中的 multiline string
playbook 是 yaml

You can format multiline strings with YAML by combining a block style indicator (| or >), a block chomping indicator (+ or –), and even an indentation indicator (1 to 9). For example, when we need a preformatted block, we use the pipe character with a plus sign (|+):

多行可以用 **| or >** 表示
**+ or –** 加上数字 表示正负缩进
```yaml
---
visiting_address: |+
Department of Computer Science
A.V. Williams Building
University of Maryland
city: College Park
state: Maryland
...
```
比如用json就没法写多行，除非你用\n

## playbook 详解
每个 playbook 必须有 hosts 变量,可以是个 group 比如 webservers, 或者特殊 group all，或者表达式

playbook 开始区域还可以定义 vars 变量，以供后续使用

## 可以从 pldaybook 中间某一步执行

```shell
ansible-playbook --start-at-task $taskName**
```


### ansible playbook 常用 module

package
    Installs or removes packages by using the host’s package manager
copy
    Copies a file from the machine where you run Ansible to the web servers
file
    Sets the attribute of a file, symlink, or directory service Starts, stops, or restarts a service
template
    Generates a file from a template and copies it to the hosts

### Ansible Module Documentation
```bash
ansible-doc service
```

### playbook string Quoting

> "The debug module will print a message: neat, eh?"
因为 ：是特殊字符需要
> '"The module will print escaped quotes: neat, eh?"'
or
> "'The module will print quoted quotes: neat, eh?'""
书中后面是两个引号需要验证1下

### Jinjia 特性
Jinjia 特性在 ansible 里面都能用
One Jinja2 feature you probably will use with Ansible is filters

### Loop

```yaml
- name: Copy TLS files
copy:
src: "{{ item }}"
dest: "{{ tls_dir }}"
mode: '0600'
loop:
    - "{{ key_file }}"
    - "{{ cert_file }}"
notify: Restart nginx
```
**{{ item }}** 就是 loop 中的list 中的两个元素的引用

### Handlers
声明：

```yaml
handlers:
- name: Restart nginx
service:
name: nginx
state: restarted
```

在task中使用
```yaml
- name: Manage nginx config template
template:
src: nginx.conf.j2
dest: "{{ conf_file }}"
mode: '0644'
notify: Restart nginx
```

handler 和 task 很像，但是只有在被task notified 的时候才用，task 只有状态时 changed的时候才会发出 notification

handler 可以在很多 task中被引用，被 notifiy，但是他们都仅仅会运行一次

如果有多个handler，那么他们是按定义时候的顺序运行的，不会按被 notifiy的顺序执行

handler 一般被用来重启 server 比如 nginx

### Testing

对于handlers,有种情况要小心：
- run playbook
- 其中一个 task 有 change state 发出 notify
- 然后某个 task 出错了， ansible 停止
- 修复后，再跑 playbook
- 当时发出 notify的 task 因为no change 不notify, handler 不运行
- 部署失败
这时候就需要在 playbook 里面家一个 test

```yaml
- name: "Test it! https://localhost:8443/index.html"
    delegate_to: localhost
    become: false
    uri:
        url: 'https://localhost:8443/index.html'
        validate_certs: false
        return_content: true
    register: this
    failed_when: "'Running on ' not in this.content"
```

### Validation
yamllint 非常有帮助

```shell
ansible-playbook --syntax-check webservers-tls.yml
ansible-lint webservers-tls.yml
yamllint webservers-tls.yml
ansible-inventory --host testserver -i inventory/vagrant.ini
vagrant validate
```