+++
title = "ansible"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 2
+++



# 安装

```bash
管理机器：
1.选择yum自动化安装，阿里云yum，epel源，前提就得配置好
yum install epel-release -y 
yum install  ansible  libselinux-python -y 
 
2.检查ansible软件安装情况，查询配置文件，和可执行命令
 
[root@m01 ~]# rpm -ql ansible | grep -E '^/etc|^/usr/bin'
 
3.检查ansible版本
ansible --version
 
4.配置ansible的配置文件，添加被管理机器的ip地址，或者主机名
cp /etc/ansible/hosts{,.ori} 
```



添加ansible需要管理的机器地址，添加主机信息 

例如：

```bash
[root@m01 ansible]# vim /etc/ansible/hosts
[any]
10.0.0.2
10.0.0.3 
```

```bash
# 被管理机器：
yum install epel-release libselinux-python -y 
 
# 管理方式：
# 传统输入ssh密码认证
# 配置ssh密钥认证
 
# 执行命令方式：
-m 指定功能模块，默认就是command模块
-a 告诉模块需要执行的参数
-k 询问密码验证
-u 指定运行的用户
# 在m01机器上，告诉其他被管理的机器，你要执行什么命令，以及用什么用户去执行
ansible chaoge  -m  command -a 'hostname' -k -u root
 
 
# 免密配置：
# 可以在 /etc/ansible/hosts文件中，定义好密码即可，即可实现快速的认证，远程管理主机
# 参数     
ansible_host   			# 主机地址
ansible_port        # 端口,默认是22端口 
ansible_user        # 认证的用户
ansible_ssh_pass    # 用户认证的密码 
```

```bash
 # 修改host配置文件如下示例：
[any]
10.0.0.2 ansible_user=root ansible_ssh_pass=111111
10.0.0.3 ansible_user=root ansible_ssh_pass=111111

# 配置好后执行ansible命令不再需要输入密码
 
# ssh密钥方式：
# 管理机器上编写公钥脚本：
#!/bin/bash
rm -rf ~/.ssh/id_rsa*
ssh-keygen -f ~/.ssh/id_rsa -P "" > /dev/null 2>&1
SSH_Pass=111111
Key_Path=~/.ssh/id_rsa.pub
for ip in 2 3
do
    sshpass -p$SSH_Pass ssh-copy-id -i $Key_Path "-o StrictHostKeyChecking=no" 10.0.0.$ip
done
# 非交互式分发公钥命令需要用sshpass指定SSH密码，通过-o StrictHostKeyChecking=no 跳过SSH连接确认信息
 
 
```



# 命令

```bash
安装：
管理端安装ansible
yum install -y epel-release  #先安装epel源
yum install -y ansible     #安装ansible
 
#配置主机之间免密登录
 
配置文件：（在/etc/anssible下）
├── roles   # 角色配置文件
├── hosts  # 主机文件
└── ansible.cfg  # 配置文件
```

 参数：

| -I             | --inventory-file=INVENTORY：指定主机清单文件路径。           |
| -------------- | ------------------------------------------------------------ |
| -m             | --module-name=MODULE_NAME：指定要使用的模块名称。            |
| -a             | --args='MODULE_ARGS'：指定命令。                             |
| -b             | --become：以超级用户（root）身份执行任务。                   |
| -K             | --ask-become-pass：在执行任务时询问超级用户密码。            |
| -t             | --tags=TAGS：只运行带有指定标签的任务。                      |
| -e             | --extra-vars=EXTRA_VARS：传递额外的变量给 Ansible Playbook。 |
| --syntax-check | 检查 playbook 的语法是否正确。                               |
| --list-hosts   | 显示在主机清单中定义的所有主机。                             |
| --list-tasks   | 显示 playbook 中定义的所有任务。                             |

在远程主机执行命令，不支持管道、重定向等shell的特性。

```bash
 ansible-doc -s command  #-s列出指定模块的描述信息和操作动作
 
 #指定ip执行date
 ansible 192.168.126.28 -m command -a 'date'  #-a指定命令
 
 #指定组执行date命令
 ansible webservers -m command -a 'date'   #指定webservers组执行date命令
 ansible all -m command -a ' date'      #all代表所有hosts 主机
 ansible all -a 'date'           #如省略-m模块，则默认运行command 模块
 
 
 ##常用的参数:
 # chdir：在远程主机上运行命令前提前进入目录
 # creates：判断指定文件是否存在，如果存在，不执行后面的操作
 # removes：判断指定文件是否存在，如果存在，执行后面的操作
 
 #无论管理端当前在哪个目录，执行命令都是在被管理端的家目录进行操作，可以使用chdir参数先切换目录
 ansible dbservers -m command -a "chdir=/home ls ./"  #切换到/home目录下再执行命令
 
 #creates判断目标主机的指定是否存在，如果存在，则不执行后面的操作
 ansible dbservers -m command -a "creates=/data/f1.txt date"
 
 #removes判断目标主机的指定是否存在，如果存在，执行后面的操作
 ansible dbservers -m command -a "removes=/data/f1.txt date"
```



# 模式

## ad-hoc模式

<img src="./images/image-20240201104902079.png" alt="image-20240201104902079" style="zoom: 80%;" />

ansible的ad-hoc模式是ansible的命令行形式，也就是处理一些临时的，简单的任务，可以直接使用ansible的命令行来操作

比如

- 临时批量查看被管理机器的内存情况，cpu负载情况，网络情况
- 比如临时的分发配置文件等等



查看该模块支持的参数：

```bash
ansible-doc -s 模块名称
```



### **command** 模块

**不支持管道重定向**

| chdir     | 在执行命令之前，先通过cd进入该参数指定的目录                 |
| --------- | ------------------------------------------------------------ |
| creates   | 在创建一个文件之前，判断该文件是否存在，如果存在了则跳过前面的东西，如果不存在则执行前面的动作 |
| free_form | 该参数可以输入任何的系统命令，实现远程执行和管理             |
| removes   | 定义一个文件是否存在，如果存在了则执行前面的动作，如果不存在则跳过动作 |

 

### shell模块

在远程机器上执行命令（复杂的命令）,支持重定向，管道

支持的参数和解释

| chdir     | 在执行命令之前，通过cd进入该参数指定的目录  ansible hosts_name -m shell -a 'chdir=/tmp ls' |
| --------- | ------------------------------------------------------------ |
| creates   | 定义一个文件是否存在，如果存在则不执行该命令，如果存在该文件，则执行shell命令 |
| free_form | 参数信息中可以输入任何的系统指令，实现远程管理               |
| removes   | 定义一个文件是否存在，如果存在该文件，则执行命令，如果不存在，则跳过 |

 

### script模块

script模块不具有幂等性。所以建议用剧本来执行

```
管理机器上的脚本在远程节点上执行
```

### serivce/systemd模块

```bash
 ansible-doc -s service
 ansible-doc -s systemd
 # 要注意的是serivce已然对centos7有效
 # 当你使用service命令管理服务，系统自动的重定向为systemctl服务管理命令
```

参数

```bash
 name 指定服务的名字，比如nginx.serivce  如 crond.serivce
 state 填入你要执行的操作，如 reloaded,restarted,started,stopped
 enabled 指定服务开机自启 systemctl enable nginx
 daemon_reload 每当修改了配置文件，使用systemd重读配置文件
```



### crond模块-定时任务服务

定时crontab条目都是遵循了规则：	

分 时 日 月 周  执行命令的绝对路径

\*   *   *   *   * 



示例：

每5分钟执行命令

```bash
*/5 * * * *  cmd xxx
```

每个月的3号，13号，早上8点整 重启nginx

```bash
0 8 3,13 * * /usr/bin/systemctl restart nginx
```

添加定时任务，每5分钟进行时间同步

```bash
ansible chaoge -m cron -a "name=ntp_cron job='/usr/sbin/ntpdate ntp.aliyun.com > /dev/null 2>&1' minute=*/5 "
```

再添加一个记录，事件是每个月的3号，13号，早上8点整 重启nginx

思路：转化如下任务即可

```bash
0 8 3,13 * * /usr/bin/systemctl restart nginx

ansible chaoge -m cron -a "name=restart_nginx job='/usr/bin/systemctl restart nginx' minute=0 hour=8 day=3,13 "
```

删除定时任务，只能删除通过ansible模块添加的任务记录

```bash
ansible chaoge -m cron -a "name='restart_nginx' state=absent" 
```

查看任务：

```bash
 ansible chaoge -m shell -a "crontab -l"
```

### **file模块**

```bash
常用的参数解释：
group  定义文件/目录的 属组
owner  定义属主
mode   定义权限
path   必选参数，定义文件路径 
src   定义源文件路径，主要用于创建link类型文件使用
dest   创建出来的软连接 它的路径
state 参数：
file ：如果目标文件不存在，那么不会创建该文件
touch： 如果文件不存在，则创建一个新的文件，如果文件已经存在了，则修它的最后修改时间
directory: 如果目录不存在，那么会创建目录
link: 用于创建软连接类型
absent : 删除目录，文件或者取消连接
```

示例：

```bash
1.远程的批量创建文件夹，并且设置权限是666
ansible chaoge -m file -a "dest=/tmp/cc_dir/ mode=666 state=directory"
 
2.验证文件夹是否存在，以及权限查看
ansible chaoge -m shell -a "ls -ld /tmp/cc_dir"
 
3.目标文件不存在，则不执行动作，这是state的file属性
ansible chaoge -m file -a "dest=/tmp/cc_666.txt state=file owner=chaoge group=chaoge mode=600"
 
4.应该使用state的touch属性
ansible chaoge -m file -a "dest=/tmp/cc_666.txt state=touch owner=chaoge group=chaoge mode=600"
```



### copy模块

1、批量拷贝文件，分发给客户端节点

```bash
ansible chaoge -m copy -a "src=/etc/hosts dest=/tmp/m01_hosts owner=learn_ansible group=learn_ansible mode=0666"
```



2、就是批量的实现了文件远程拷贝，且定义了新的内容放入文件中，并且真对目标机器的源数据文件，做了一个备份

```bash
ansible chaoge -m copy -a "content='Hello,my name is chaoge,who are u' dest=/tmp/day.txt backup=yes"
```



### user模块

**模块的各个参数说明：**

- name（必需）： 要操作的用户的名称。
- uid： 用户的 UID（用户标识号）。
- group： 用户所属的用户组。
- groups： 用户所属的其他附加用户组列表。
- append： 如果用户已存在，是否将用户添加到指定的用户组中。
- state： 用户的状态，可以是 present（用户存在）、absent（用户不存在）、locked（用户被锁定）等。
- createhome： 如果用户不存在，是否创建用户的 home 目录。
- home： 用户的 home 目录路径。
- shell： 用户的默认 shell。
- comment： 用户的注释信息。
- expires： 用户账号的过期时间。
- password： 用户的密码（加密后的形式，可以使用 ansible-vault 进行加密）。
- update_password： 指定密码的更新策略。
- move_home： 如果用户的 home 目录发生变化，是否将用户的文件一起移动。

 

### yum模块

```bash
1、通过yum模块给any主机组批量安装服务
ansible any -m yum -a "name=nginx state=installed"
 
2.any主机组批量远程卸载nginx
ansible any -m yum -a "name=nginx state=absent"
 
3.升级软件包，指定升级nginx，也可以写成 name='*' 就等于 yum update升级所有软件包，latest也提供下载更新
ansible chaoge -m yum -a "name='nginx' state=latest"
 
4.升级系统所有软件包，排除某个服务不升级，这个命令，注意不要在服务器上随便敲，因为服务器不得任意更新一些服务版本，可能会造成服务挂掉
ansible chaoge -m yum -a "state=latest name='*' exclude='nginx'"
```



## playbook模式
ansible的playbook模式是针对比较具体，且比较大的任务，那么你就得实现写好剧本，应用场景

- 一键部署rsync备份服务器

- 一键部署lnmp环境

 

参考博客：有高阶使用
https://www.cnblogs.com/yanjieli/p/10969299.html

 

参数：

| -i                     | INVENTORY,  --inventory=INVENTORY：指定主机清单文件路径。    |
| ---------------------- | ------------------------------------------------------------ |
| -e                     | EXTRA_VARS,  --extra-vars=EXTRA_VARS：传递额外的变量给 Ansible Playbook。 |
| --list-tasks           | 显示 playbook 中定义的所有任务。                             |
| --syntax-check         | 检查 playbook 的语法是否正确。                               |
| --start-at-task='TASK' | 从指定任务开始执行 playbook。                                |

### 执行命令

```bash
# 编写执行：
ansible-playbook xxxx.yaml

# 查看详细输出：
ansible-playbook xxxx.yaml --verbose

# 查看剧本中主机信息：
ansible-playbook xxxx.yaml --list-hosts

# 执行剧本加载指定的主机清单文件，默认剧本使用的是 /etc/ansible/hosts
ansible-playbook xxxx.yaml -i /etc/my_ansible/hosts

# 执行剧本并且检查语法
ansible-playbook xxxx.yaml --syntax-check

# 调试剧本，只是调试，但是不会对被管理节点发生改变
ansible-playbook nginx.yaml -C # 模拟执行，不影响客户端机器
```

### host部分配置

```bash
# 方式一，定义被管理主机的ip地址
- hosts: 192.168.178.138
 tasks:
  - name: 这是我第一个任务
   yum: name=nginx state=installed
 
# 方式二，定义被管理主机的名字，注意该主机名必须能够解析
- hosts: backup01
 tasks:
  - name: 这是我第一个任务
  
# 方式三，定义多个主机信息
- hosts: 192.168.178.138,192.168.178.139,backup01
 tasks:
  - name: 这是我第一个任务
  
# 方式四，填写所有的主机
- hosts: all
 tasks: 
 
```

### task部分配置

```bash
#定义task方式一：采用变量形式设置任务
tasks:
 - name: make sure apache is runing
  service: name=httpd state=running
  
 - name: copy file..
  copy: src=/etc/ansible/hosts dest=/etc/ansible/hosts
     owner=root group=root mode=0644 
 
#当传入的参数列表过长的时候，我们还可以将其分割
 
# 方式二，采用字典形式设置多个任务
tasks:
 - name: copy file to client...
  copy:
   src: /etc/ansible/hosts
   dest: /etc/ansible/hosts
   owner: root
   group: root
   mode: 0644
```

### 示例

#### 安装nginx

```bash
- hosts: any
 tasks:
  - name: 安装nginx
   yum: name=nginx state=installed
      - name: 执行脚本
       script: /server/scripts/test_ansible.sh
```



#### 部署rsync服务

```bash
[root@anyl]# cat install_rsync.yaml
- hosts: hosts_name
 tasks:
  - name: step01,install rsync service
   yum: name=rsync state=installed
 
  - name: step02,edit rsync conf file
   copy: src=/etc/ansible/rsync_conf/rsyncd.conf dest=/etc/rsync/conf/
 
  - name: step03,create user rsync
   user: name=rsync state=present createhome=no shell=/sbin/nolgoin
 
  - name: step04,create user auth file
   copy: src=/etc/ansible/rsync_conf/rsync.password dest=/etc/rsync/conf/ mode=0600
 
  - name: step05,create backup dir
   file: dest=/data_backup/ state=directory owner=rsync group=rsync
 
  - name: step06,run rsync server
   shell: rsync --daemon creates=/var/run/rsync.pid
```



#### 指定变量

```bash
# cat variables.yml  --- - hosts: all  remote_user: root
tasks:   - name: install pkg    yum: name={{ pkg }} state=install
 
# 执行：传参
ansible-playbook -e "pkg=nginx"variables.yml
```



#### host文件中定义变量

```bash
# 编辑hosts文件定义变量
vim /etc/ansible/hosts
[apache]
192.168.1.36 webdir=/opt/test   #定义单个主机的变量
192.168.1.33
[apache:vars]   #定义整个组的统一变量
webdir=/web/test
 
[nginx]
192.168.1.3[1:2]  #表示ip为192.168.1.31 192.168.1.32
[nginx:vars]
webdir=/opt/web
 
 
# 编辑playbook文件
# cat variables.yml 
---
- hosts: all
 remote_user: root
 
 tasks:
  - name: create webdir
   file: name={{ webdir }} state=directory  #引用变量
 
# 执行playbook
# ansible-playbook variables.yml
```

#### playbook文件中定义变量

```bash
# 编辑playbook
 cat variables.yml 
---
- hosts: all
 remote_user: root
 vars:        #定义变量
  pkg: nginx     #变量1
  dir: /tmp/test1  #变量2
 
 tasks:
  - name: install pkg
   yum: name={{ pkg }} state=installed  #引用变量
  - name: create new dir
   file: name={{ dir }} state=directory  #引用变量
 
 
# 执行playbook
 ansible-playbook variables.yml
 
# 如果执行时候又重新指定了变量的值，那么会已重新指定的为准
 ansible-playbook -e "dir=/tmp/test2" variables.yml
```



#### 独立的变量yaml文件中定义

```bash
# 定义存放变量的文件
cat var.yml 
var1: vsftpd
var2: httpd
 
# 编写playbook
cat variables.yml 
---
- hosts: all
 remote_user: root
 vars_files:  #引用变量文件
  - ./var.yml  #指定变量文件的path（这里可以是绝对路径，也可以是当前yaml文件的相对路径）
 
 tasks:
  - name: install package
   yum: name={{ var1 }}  #引用变量
  - name: create file
   file: name=/tmp/{{ var2 }}.log state=touch  #引用变量
 
# 执行playbook
ansible-playbook variables.yml
```



#### playbook标签的使用

```bash
# 编辑playbook
cat httpd.yml 
---
- hosts: 192.168.1.31
 remote_user: root
 
 tasks:
  - name: install httpd
   yum: name=httpd state=installed
   tags: inhttpd
 
  - name: start httpd
   service: name=httpd state=started
   tags: sthttpd
 
  - name: restart httpd
   service: name=httpd state=restarted
   tags: 
    - rshttpd
    - rs_httpd
 

# 正常执行，会全部执行
ansible-playbook httpd.yml 
 
# 通过-t指定tags名称，多个tags用逗号隔开
ansible-playbook -t rshttpd httpd.yml 
 
#通过--skip-tags选项排除不执行的tags
ansible-playbook --skip-tags inhttpd httpd.yml 
 
 
```

# roles

## 介绍

 ansible自1.2版本引入的新特性，用于层次性、结构化地组织playbook。
​ roles能够根据层次型结构自动装载变量文件、tasks以及handlers等。`​`要使用roles只需要在playbook中使用include指令即可。

简单来讲，roles就是通过分别将变量、文件、任务、模板及处理器放置于单独的目录中，

复杂场景：建议使用roles，代码复用度高
变更指定主机或主机组
如命名不规范维护和传承成本大
某些功能需多个Playbook，通过includes即可实现

```bash
角色(roles)：角色集合
[root@ansible ansible]# tree 
.
└── roles
    ├── httpd
    ├── memcache
    ├── mysql
    └── nginx
    
可以互相调用
```

roles各目录作用

```bash
/roles/project/ :项目名称,有以下子目录
    files/ ：存放由copy或script模块等调用的文件
    templates/：template模块查找所需要模板文件的目录
    tasks/：定义task,role的基本元素，至少应该包含一个名为main.yml的文件；
            其它的文件需要在此文件中通过include进行包含
    handlers/：至少应该包含一个名为main.yml的文件；
               其它的文件需要在此文件中通过include进行包含
    vars/：定义变量，至少应该包含一个名为main.yml的文件；
           其它的文件需要在此文件中通过include进行包含
    meta/：定义当前角色的特殊设定及其依赖关系,至少应该包含一个名为main.yml的文件，
           其它文件需在此文件中通过include进行包含
    default/：设定默认变量时使用此目录中的main.yml文件
    
roles/appname 目录结构
    tasks目录：至少应该包含一个名为main.yml的文件，其定义了此角色的任务列表；
               此文件可以使用include包含其它的位于此目录中的task文件
    files目录：存放由copy或script等模块调用的文件；
    templates目录：template模块会自动在此目录中寻找Jinja2模板文件
    handlers目录：此目录中应当包含一个main.yml文件，用于定义此角色用到的各handler；
                  在handler中使用include包含的其它的handler文件也应该位于此目录中；
    vars目录：应当包含一个main.yml文件，用于定义此角色用到的变量；
    meta目录：应当包含一个main.yml文件，用于定义此角色的特殊设定及其依赖关系；
              ansible1.3及其以后的版本才支持；
    default目录：为当前角色设定默认变量时使用此目录；应当包含一个main.yml文件

roles/example_role/files/             所有文件，都将可存放在这里
roles/example_role/templates/         所有模板都存放在这里
roles/example_role/tasks/main.yml：   主函数，包括在其中的所有任务将被执行
roles/example_role/handlers/main.yml：所有包括其中的 handlers 将被执行
roles/example_role/vars/main.yml：    所有包括在其中的变量将在roles中生效
roles/example_role/meta/main.yml：    roles所有依赖将被正常登入

```

## 创建roles

```bash
创建role的步骤
(1) 创建以roles命名的目录
(2) 在roles目录中分别创建以各角色名称命名的目录，如webservers等
(3) 在每个角色命名的目录中分别创建files、handlers、meta、tasks、templates和vars目录；
    用不到的目录可以创建为空目录，也可以不创建
(4) 在playbook文件中，调用各角色
```

示例：创建一个nginx role

```bash
# 1.创建roles目录
   mkdir roles/{httpd,mysql,redis}/tasks -pv
   mkdir  roles/httpd/{handlers,files}
   
# 2.查看目录结构
[root@ansible ansible]# tree  roles/
roles/
└── nginx
    ├── tasks
    └── templates
        
# 3.创建目标文件
[root@ansible ansible]#touch group.yaml user.yaml yum.yaml templates.yaml start.yaml
# vim group.yaml 
- name: create group 
  group: name=nginx 

# vim  user.yaml 
- name: create user 
  user: name=nginx group=nginx  shell=/sbin/nologin

# vim  yum.yaml 
- name: copy file 
  copy: src=/etc/yum.repos.d/CentOS-Base.repo dest=/etc/yum.repos.d/CentOS-Base.repo
- name: install packages 
  yum: name=nginx 

# vim templates.yaml 
- name:  copy conf 
  template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf 

# vim start.yaml 
- name: start service 
  service: name=nginx state=started enabled=yes

# 4.创建main.yml主控文件,调用以上单独的yml文件,main.yml定义了谁先执行谁后执行的顺序
[root@ansible tasks]# vim main.yaml 
- include: group.yaml 
- include: user.yaml 
- include: yum.yaml 
- include: templates.yaml
- include: start.yaml 

# 5.创建好j2的nginx配置文件
# vim templates/nginx.conf.j2 
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes {{ansible_processor_vcpus+2}};  #改个CPU数量
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# 6.在roles同级的情况下创建使用角色role的yaml文件。
[root@ansible ansible]# vim nginx_role.yaml  
- hosts: server 
  remote_user: root 
  roles: 
    - role: nginx  #调用角色

# 7.查看完整的目录结构
[root@ansible ansible]# tree  roles 
roles
├── httpd
├── memcache
├── mysql
└── nginx
    ├── tasks
    │   ├── group.yaml
    │   ├── main.yaml
    │   ├── start.yaml
    │   ├── templates.yaml
    │   ├── user.yaml
    │   └── yum.yaml
    └── templates
        └── nginx.conf.j2

# 8.运行role的剧本
[root@ansible ansible]# ansible-playbook nginx_role.yaml 

PLAY [server] **********************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [192.168.1.107]
ok: [192.168.1.106]
ok: [192.168.1.105]

TASK [nginx : create group] ********************************************************************************************
ok: [192.168.1.105]
ok: [192.168.1.107]
ok: [192.168.1.106]

TASK [nginx : create user] *********************************************************************************************
ok: [192.168.1.105]
ok: [192.168.1.107]
ok: [192.168.1.106]

TASK [nginx : copy file] ***********************************************************************************************
ok: [192.168.1.107]
ok: [192.168.1.105]
ok: [192.168.1.106]

TASK [nginx : install packages] ****************************************************************************************
ok: [192.168.1.107]
ok: [192.168.1.106]
ok: [192.168.1.105]

TASK [nginx : copy conf] ***********************************************************************************************
ok: [192.168.1.105]
ok: [192.168.1.106]
ok: [192.168.1.107]

TASK [nginx : start service] *******************************************************************************************
changed: [192.168.1.106]
changed: [192.168.1.107]
changed: [192.168.1.105]

PLAY RECAP *************************************************************************************************************
192.168.1.105              : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.106              : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.107              : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 

```

## 多角色调用

```bash
# 调用角色方法1：
[root@ansible ansible]# vim web.yaml 
- hosts: server 
  remote_user: root 

  roles: 
    - role: httpd 
    - role: nginx

# 调用角色方法2
# 传递变量给角色
- hosts:
  remote_user:
  roles:
    - mysql
    - { role: nginx, username: nginx }   #不同的角色调用不同的变量  
    # 键role用于指定角色名称
    # 后续的k/v用于传递变量给角色

# 调用角色方法3：还可基于条件测试实现角色调用
roles:
  - { role: nginx, username: nginx, when: ansible_distribution_major_version == '7' }
```

## 完整roles架构

```bash
cat nginx-role.yml 顶层任务调用yml文件
---
- hosts: testweb
  remote_user: root
  roles:
    - role: nginx
    - role: httpd 可执行多个role

cat roles/nginx/tasks/main.yml
---
- include: groupadd.yml
- include: useradd.yml
- include: install.yml
- include: restart.yml
- include: filecp.yml

cat roles/nginx/tasks/groupadd.yml
---
- name: add group nginx
  user: name=nginx state=present

cat roles/nginx/tasks/filecp.yml
---
- name: file copy
  copy: src=tom.conf dest=/tmp/tom.conf

# 以下文件格式类似：
useradd.yml,install.yml,restart.yml

ls roles/nginx/files/
tom.conf
```

## 通过roles传递变量

```bash
通过roles传递变量
当给一个主机应用角色的时候可以传递变量，然后在角色内使用这些变量
示例：
- hosts: webservers
  roles:
    - common
    - { role: foo_app_instance, dir: '/web/htdocs/a.com', port: 8080 }
```

## 向role传递变量

```bash
```

```yaml
而在playbook中，可以这样使用roles：
---
- hosts: webservers
  roles:
    - common
    - webservers

也可以向roles传递参数
示例：
---
- hosts: webservers
  roles:
    - common
    - { role: foo_app_instance, dir: '/opt/a', port: 5000 }
    - { role: foo_app_instance, dir: '/opt/b', port: 5001 }
```

## 条件式地使用roles

```yaml
甚至也可以条件式地使用roles
示例：
---
- hosts: webservers
  roles:
    - { role: some_role, when: "ansible_os_family == 'RedHat'" }
```

## Roles条件及变量等案例

```yaml
When条件
    roles:
      - {role: nginx, when: "ansible_distribution_major_version == '7' " ,username: nginx }
变量调用
- hosts: zabbix-proxy
  sudo: yes
  roles:
    - { role: geerlingguy.php-mysql }
    - { role: dj-wasabi.zabbix-proxy, zabbix_server_host: 192.168.37.167 }
```

## roles playbook tags使用

```yaml
roles playbook tags使用
    ansible-playbook --tags="nginx,httpd,mysql" nginx-role.yml  对标签进行挑选执行

// nginx-role.yml
---
- hosts: testweb
  remote_user: root
  roles:
    - { role: nginx ,tags: [ 'nginx', 'web' ] ,when: ansible_distribution_major_version == "6“ }
    - { role: httpd ,tags: [ 'httpd', 'web' ] }
    - { role: mysql ,tags: [ 'mysql', 'db' ] }
    - { role: marridb ,tags: [ 'mysql', 'db' ] }
    - { role: php }
```

## 示例

```bash
memcacched 当做缓存用,会在内存中开启一块空间充当缓存
cat /etc/sysconfig/memcached 
    PORT="11211"
    USER="memcached"
    MAXCONN="1024"
    CACHESIZE="64"    # 缓存空间默认64M 
    OPTIONS=""

1> 创建对用目录
   cd /app/ansible
   mkdir roles/memcached/{tasks,templates} -pv
   
2> 拷贝memcached配置文件模板
   cp /etc/sysconfig/memcached  templates/memcached.j2
   vim templates/memcached.j2
   CACHESIZE="{{ansible_memtotal_mb//4}}"   #物理内存的1/4用做缓存
   
3> 创建对应yml文件,并做相应配置
   cd tasks/
   touch install.yml config.yml service.yml
   创建main.yml文件定义任务执行顺序
   vim main.yml
   - include: install.yml
   - include: config.yml
   - include: service.yml  
   
   vim install.yml
   - name: install 
     yum: name=memcached
     
   vim config.yml
   - name: config file
     template: src=memcached.j2 dets=/etc/sysconfig/memcached

   vim service.yml
   - name: service
     service: name=memcached state=started enabled=yes

4> 创建调用角色文件
   cd /app/ansible/roles/
   vim role_memcached.yml
    ---
    - hosts: appsrvs
    
      roles: 
        - role: memcached

5> 安装
   ansible-playbook  role_memcached.yml 
   memcached端口号11211
```

