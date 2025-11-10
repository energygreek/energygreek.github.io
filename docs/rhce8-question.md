---
date: 2020-08-29
tags: [rhce,linux]
---

# 记录一下rhce8上课笔记

1. 进入nfs，autofs目录卡住，如何退出，应该是进入了D状态 
2. chrome 需要笔记本插件，js控制插件
3. 修改root密码时，在linux 选项后面添加 rd.break 然后默认进入内核根目录，然后rw重新挂载系统根目录 /sysroot/ 
4. fdisk 设置分区类型: 在创建分区结束后，使用`t`来标记文件类型, 然后使用`L`查看类型代码，查找到Linux swap 对应的代码是85


## RHCIA

### 修改了网络信息之后，必须使用nmcli connection down和up connection才会生效
```
nmtui connection down Wired_connection1
nmtui connection up Wired_connection1
```

### 配置httpd服务时，修改port后，因为port非标准端口，需要用semanage打tag。否则失败，可以查看journeyctl -e 查看到错误信息和解决办法

```
semanage port -a -t http_port_t -p tcp 82
```


### 创建/home/admins 目录，并设置在目录中创建的文件，其组所有权自动设置为 adminusr。 这里就是设置s位，有数值方法和+-法
```
chmod 2770 /home/admins

# 或

chmod o-x /home/admins
chmod o-r /home/admins
chmod g+w /home/admins
chmod g+s /home/admins # 这里设置组的s位
```

验证方法，进入目录创建文件，查看组名

### autofs 挂载有两个关键，第一在/etc/auto.master.d/ 目录下创建后缀名为autofs的文件， 第二是编辑挂载配置文件
内容格式
```
# cat /etc/auto.master.d/indirect
#主挂载点	挂载配置
/internal	/etc/auto.demo
```

内容配置，* 表示所有目录（根据nfsserver的情况）， 和&搭配使用
```
# cat /etc/auto.demo
# 间接挂载目录	挂载配置	挂载路径（nfs地址）
*	-rw,fstype=nfs4,sync	servera:/shares/indirect/&
```

### setfacl 和getfacl 设置和显示文件属性

```
设置natash用户读写权限
setfacl u:natasha:rw	/var/tmp/fstab
设置harry用户没有任何权限 
setfacl u:harry:-	/var/tmp/fstab
```

### 重置root密码，在linux 行末追加 rd.local, 按ctrl+x 启动
重新挂载系统根目录, chroot到根目录和修改密码
```
mount -orw,remount /systemroot/
chroot /systemroot/
passwd
```

### 逻辑卷扩容

设置分区start位置的时候，输入2048s。 s指扇区，一个扇区为512字节， 2048s指1M，这1M的分区为esp分区

扩容命令lvextrend，这个扩容只是将逻辑卷扩容， 而题目要求对文件系统扩容， 还需要执行文件系统类型对应的扩容命令如resize2fs(ext), xfs_growfs(xfs)。 
但对lvextend命令添加 -r 参数可以一步到位
```
# 扩容到300MB
lvextend -r -L 300MB /dev/new_redhat/lv1
```

### 创建逻辑分区, 注意避免磁盘不够用，尽量共用上一题的磁盘。而且创建卷组的时候不要忘记-s参数

题目可能不是直接给分区大小，而是给出扩展块大小和拓展块个数，其实分区大小=拓展块大小 × 拓展块个数。 在lvcreate -l 来指定块个数 

这时必须使用msdos分区类型才有主分区和拓展分区，而且为拓展分区分配剩余的所有空间。 最后在拓展分区上创建逻辑分区， 并将创建的逻辑分区加入卷组

创建分区，大小通过上面的算法计算出来
```
parted 
mkpart
print
exit
```

创建卷组
```
vgcreate -s 20MB -n npgroup /dev/vdb2
```
创建逻辑卷, 注意是小写l
```
lvcreate -l 45 -n np npgroup
```
这样就创建了一个 20×45 = 900MB的逻辑卷

### 创建新的swap分区，注意需要修改fstab, 而且验证的时候，不需要重启系统，而是执行  
```
systemctl daemon-reload 
swapon -a  # 类似mount -a
swapon --show 
```

### vdo  --vdoLogicalSize 5G, 不能使用GB

```
vdo create --name test --vdoLogicalSize 5G --device /dev/vdb
```
`blkid` 查到 UUID, 最后添加到fstab

### vdo 持久挂载 mount
```
default,x-systemd.requires=vdo.service
```

## RHCE

ansible的有用命令
ansbile-doc -l | grep  查找模块和介绍
ansible-doc 模块 查看模块具体说明

### 创建inventory 文件， 子组的写法为
[webservices:children]
prod

prod 是webserverices组的子组

### 创建ansible.cfg 文件，可以复制 /etc/ansbile/ansible.cfg 到～/ansbile/目录下

容易忘记的选项是远程登陆身份， 即配置 remote_user = devops
如果环境没有配置免密登陆，需要在[defaults] 栏添加 `ask_pass = True`

验证方法 `ansible all -m ping`

### ansbile ad-hoc 添加yum 源  ansible 模块的使用方法 ansible 主机名 -m 模块 -a '参数'

脚本要添加可执行权限
 
```sh
#!/bin/bash

# 先要切换到工作目录
cd /home/devops/ansible/

ansible all -m yum-repository  -a 'name=  description=  baseurl=  enabled= gpgcheck= gpgkey= '
```

### 安装软件包

安装软件包如'Development Tools', 需要在包名前面加'@'即 '@Development Tools'
验证yaml 语法格式: `ansible-playbook --syntax-check *.yml`

```yaml
- name: install  development tools
  hosts: dev
  tasks:
    - name: install development tools
        yum:
          name: "@Development Tools"
          state: present                  
```

### 同步时间依赖角色，安装角色包'rhel-system-roles' 需要在配置文件里添加上roles路径

playbook 示例在/usr/share/ansibe/roles/rhel-system-roles.timesync/Readme.md
现成的yml示例在/usr/share/doc/rhel-system-roles/timesync/example-timesync-playbook.yml

```
# ansible.cfg
[default]
roles_path = /home/devops/ansible/roles:/usr/share/ansible/roles
```

### 使用ansible-galaxy 命令安装角色，安装到本地roles目录

yml文件很简单, 执行命令 ansible-galaxy install -r roles/requirement.yml -p roles/
-p 表示安装路径 

```yml
- name: balancer
  file: /home/devops/ansible/files/5/haproxy.tar
```

### 创建apache角色， 角色对应一系列的任务和模板文件

通过`ansible-galaxy init apache` 来初始化角色目录，修改角色目录下的tasks/mail.yml 来添加任务，任务所需的模板文件在同级目录templates下  
通过`ansible servera -m setup` 来查询环境变量， 在模板文件中使用`ansible_facts['fqdn']` 来引用变量
题目说的hostname 是`serverc.lab.example.com` 对应变量是fqdn, 另外一个是ip即 ['default_ipv4']['address']

例如，安装httpd用到yum模块， 启用httpd 用到service模块， 防火墙配置用到firewalld模块
```yml
# home/devops/ansible/role/apache/tasks/main.yml

---
- name: install httpd
  yum:
    name: httpd
    state: present

- name: enable httpd
  service:
    name: httpd
    state: started
    enabled: yes
    
- name: firewalld config
  firewalld:
    service: httpd
    immediate: yes
    permanent: yes
    state: enabled
   
- name: install templates
  template:
    src: index.html.j2
    dest: /var/www/html/index.html 
```

模块文件放到templates目录下
```
# home/devops/ansible/role/apache/templates/index.html.j2
Welcome to {{ ansible_facts['fqdn'] }} on {{ ansible_facts['default_ipv4']['address'] }}

```

最后还需要创建一个playbook, 如同其他playbook 一样执行
```yml
# newrole.yml
---
- name: install new role
  hosts: webservers
  roles:
    - apache
```

### 执行balancer 角色, 创建一个负载均衡

验证：
ssh到serverd 上，curl localhost, 先是访问的serverb, 然后访问的serverc

```yml
---
- name: deploy apache on webservers
  hosts: webservers
  roles:
    - apache

- name: deploy balancer on balancer
  hosts: balancers
  roles:
    - balancer
    
- name: deploy phpinfo on webservers
  hosts: webservers
  roles:
    - phpinfo

```

#### 可能出现的错误，提示应该使用bind，这是由于balance role的template 错误
修改办法, 将listen行换行并加上bind

```diff
# /home/devops/ansible/roles/balancer/templates/haproxy.cfg.j2

backend app
  {% for host in groups.balancers %}
-  	listen {{ daemonname }} 0.0.0.0:{{ listenport }}
+  	listen {{ daemonname }} 
+	bind 0.0.0.0:{{ listenport }}
  {% endfor %}
```

### 创建逻辑卷，考验block/rescue/always, 使用lvol，`when` 和play对齐

使用了block/rescue/always就不需要tasks关键词

```yml
---
- name: create a logic volume with size of 5000 and format
  hosts: all
  block:
    - name: create logic volume
	  lvol:
	    vg: research
	    lv: data
	    size: 1500
    - name: fomat data volume
        filesystem:
          fstype: ext4
          dev: /dev/research/data
  rescue:
    - name: debug message
      debug:
        msg: can't create a size of 1800
    - name: create a logical volume with size of 800 and format
        lvol:
          vg: research
          lv: data
          size: 1500
    - name: fomat data volume
        filesystem:
          fstype: ext4
          dev: /dev/research/data
      when: ansible_facts['lvm']['vgs']['research'] is defined
      ignore_errors: yes
    - name: debug 2 
        debug:
          msg: volume group does not exist
      when: ansible_facts['lvm']['vgs']['research'] is undefined
```

### 创建hosts文件， 考验jinija 的for循环和读inventory文件

取facts变量的方法参考balancer的模板： 
/home/ansible/roles/balancer/roles/template/haproxy.cfg.j2

基于给的hosts.j2 文件修改, 拼凑出`ip url alias`格式，分别对应 default_ipv4.address, fqdn, hostname  
ansible_facts在hostvars[host]中

```yml
# 取
{% for host in groups.all %}
{{ hostvars[host]['ansible_facts'] *忽露不写了* }} × × 
{% endfor %}
--- 

创建playbook的要小心，题目要求生成所有主机的纪录，所以hosts是all, 但只部署在dev组的主机上要加when,   
要加`when: 'dev' in group_names`

```yml
---
- name: gen hosts and deploy on dev group
  tasks:
    - name: gen hosts and deploy
        template:
          src: hosts.j2
          dest: /etc/myhosts
      when: "'dev' in group_names"
```

### 创建web server, 注意将文件标注setype为httpd_system_content_t

通过man semanager-fcontext 查询setype, 或者使用`ls -ldZ`来查询selinux标签

创建文件，目录、链接都用file模块，但操作文件内容用copy模块
创建组用group模块,创建用户用user模块
容易搞反的链接的src是指向的文件，所以是/webdev, 可以看帮助文档

```diff
---
- name: create web server
  hosts: webservers
  tasks:
    - name: add group
        group:
           name: webdev
           state: present
    - name: create directory
        file:
          path: /webdev
          group: webdev
          state: directory
          mode: u+rwx,g+rwxs,o-rx
 -        setype: httpd_system_content_t
    - name: create a symbolic link
        file:
 -        src: /var/www/html/webdev
 -        dest: /webdev
 +        src: /webdev
 +        dest: /var/www/html/webdev
          state: link
    - name: create index.html
        copy:
          content: Development
          dest: /var/www/html/webdev/index.html
+         setype: httpd_system_content_t                    
```

### 创建硬件报告，用到loops进行数组循环，内置变量item和 regexs匹配，lineinfile 进行文件编辑

```yml
---
- name: generate hw report
  hosts: all
  vars:
    hw_all:
      - hw_name: HOST
        hw_content: "{{ ansible_facts['hostname'] }}"
      - hw_name: BIOS_version
        hw_content: "{{ ansible_facts['bios_version'] }}"
```

### 创建加密文件，用到ansible-vault命令

```
ansible-vault encrypt lock.yml --vault-password-file=secret.txt 
```

### 添加用户， ansible-playbook 通过 --vault-password-file=secret.txt 来传密码文件

通过`vars_files`来读取变量文件， loop 来循环遍历数组变量， 然后模板直接取密码后用'| password_hash('sha512')' 来加密密码
通常情况下在playbook中，开头位置引用一个变量时都需要"{{ variable }}"， 但是在when块里，则直接使用 variable。 另外字符串要用“string”  

创建用户时， 指定附属组用groups, 主要组是group

```
- {{ variable }} ## 错误
key: {{ variable }} ## 错误
- "{{ variable}}" ## 正确
key: "{{ variable }}" ## 正确
```

结果
```diff
---
- name: create user on dev,test
  hosts: dev,test
  vars_files:
    - lock.yml
    - user_list.yml
  tasks:
    - name: create group devops
      group:
        name: devops
        state: present
    - name: create user 
      user:
        name: "{{ item.name }}"
-       group: devops
+       groups: devops        
        password: "{{ pw_developer | password_hash('sha512') }}"
      loop: "{{ user }}"
-     when: "{{ item.job }}" == developer
+     when: item.job == "developer"
- name: create user on prod
  hosts: prod
  vars_files:
    - lock.yml
    - users_list.yml
  tasks:
    - name: create group manager
      group:
        name: manager
        state: present
    - name: create user
      user:
        name: "{{ item.name }}"
        groups: manager
        password: "{{ pw_manager | password_hash('sha512') }}"
      loop: "{{ user }}"
-     when: "{{ item.job }}" == manager
+     when: item.job == "manager"
```
      
### 修改密码, 使用`ansible-vault rekey` 修改密码，或者先解密后加密

### 修改文件内容, 使用copy模块，以及when条件判断

在playbook的内建变量 inventory_hostname 表示定义在inventory里的主机名称  
也可以使用ansible_facts['hostname'],前提是被管理节点的hostname必须如果管理节点定义名称一样。 因为后者是到被管理节点获取。

使用when 限定条件， 满足条件才能执行这个task
```yml
---
- name: create issue
  hosts: all
  tasks:
    - name: create issue file
      copy:
        content: "Development"
        dest: /etc/issue
      when: inventory_name in groups.dev  
```      

### 第二次排错

#### ansible-galaxy 安装角色, 指定方法分别是 src/name

```yml
- src: files/5/balancer.tar
  name: balancer
- src: files/5/phpinfo.tar
  name: phpinfo
```

### timesync 看doc 示例， 验证方法是看timedatectl 看system clock syncronize 为 yes

```yml
---
- name: use timesync role
  hosts: all
  vars:
    timesync_ntp_servers:
      - hostname: 
        iburst: yes
  roles:
    - rhel_system_roles.timesync
```

### 创建hosts, 要用 {% for  host in groups.all %} {{ hostvars[host].ansible_facts }} {% endfor %}


### 使用firewalld模块时，需要指定immediate 参数，确保能立即生效

### 设置文件的权限时，应使用u=rwx,g=rwxs,o=rw, 而不是u+rwx，原因很好理解
