+++
title = "gitlab"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 2
+++

# 安装

环境硬件要求：最低2核4GB，建议4核8GB（比较重）

## yum安装

```bash
# 配置源
cat >/etc/yum.repos.d/gitlab-ce.repo <<EOF
[gitlab-ce]
name=Gitlab CE Repository
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el$releasever/
gpgcheck=0
enabled=1
EOF

yum makecache
```

```bash
# 可以访问"https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/"查看Gitlab-ce的版本。
yum install -y gitlab-ce
# 指定版本
yum install -y gitlab-ce-{VERSION}
```

配置文件

建议使用HTTPS

```bash
[root@localhost ~]# vim /etc/gitlab/gitlab.rb
######### 基础配置 #########
#用户访问所使用的URL，域名或者IP地址
external_url 'http://gitlab.example.com/'
#时区
gitlab_rails['time_zone'] = 'Asia/Shanghai'

######### SSH配置 #########
gitlab_rails['gitlab_shell_ssh_port'] = 10222
#使用SSH协议拉取代码所使用的连接端口。

######### 邮箱配置 #########
gitlab_rails['smtp_enable'] = true
#启用SMTP邮箱功能，绑定一个第三方邮箱，用于邮件发送
gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
#设置SMTP服务器地址
gitlab_rails['smtp_port'] = 465
#设置SMTP服务器端口
gitlab_rails['smtp_user_name'] = "xxx@xxx.cn"
#设置邮箱账号
gitlab_rails['smtp_password'] = "xxx"
#设置邮箱密码
gitlab_rails['smtp_authentication'] = "login"
#设置邮箱账号密码身份验证方式，"login"表示采用账号密码的方式登陆
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
#设置开启SMTP邮件使用TLS传输加密协议传输邮件，以保证邮件安全传输
gitlab_rails['gitlab_email_from'] = 'xxx@xxx.cn'
#设置Gitlab来源邮箱地址，设置登陆所使用的邮箱地址

######### WEB配置 #########
nginx['enable'] = true
#启用Nginx服务
nginx['client_max_body_size'] = '250m'
#设置客户端最大文件上传大小
nginx['redirect_http_to_https'] = true
#设置开启自动将HTTP跳转到HTTPS
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.xxx.cn.pem"
#设置HTTPS所使用的证书
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.xxx.cn.key"
#设置HTTPS所使用的证书密码
nginx['ssl_protocols'] = "TLSv1.1 TLSv1.2"
#设置HTTPS所使用的TLS协议版本
nginx['ssl_session_cache'] = "builtin:1000  shared:SSL:10m"
#设置开启SSL会话缓存功能
nginx['ssl_session_timeout'] = "5m"
#设置SSL会话超时时间
nginx['listen_addresses'] = ['*', '[::]']
#设置Nginx监听地址，"*"表示监听主机上所有网卡的地址
nginx['gzip_enabled'] = true
#设置开启Nginx的传输压缩功能，以节约传输带宽，提高传输效率
```

启动

```bash
# 当配置文件发生变化时，或者是第一次启动时，我们需要刷新配置。
systemctl restart gitlab-runsvdir
gitlab-ctl reconfigure

# 启动服务
gitlab-ctl restart
# 查看状态
gitlab-ctl status
```

根据配置文件种external_url的配置，访问web界面

第一次需要输入新的超级管理员（root）密码。修改成功后，我们使用超级管理员用户“root”账号登录Gitlab管理平台。

## rpm离线

安装文档：https://docs.gitlab.com/omnibus/installation/
rpm软件包地址：https://packages.gitlab.com/gitlab/gitlab-ce
国内下载地址：https://mirror.tuna.tsinghua.edu.cn/help/gitlab-ce/

```bash
wget --content-disposition https://packages.gitlab.com/gitlab/gitlab-ce/packages/el/8/gitlab-ce-15.10.2-ce.0.el8.x86_64.rpm/download.rpm

yum -y install gitlab-ce-16.10.2-ce.0.el8.x86_64.rpm

vim /etc/gitlab/gitlab.rb # 编辑站点地址
32 external_url 'http://192.168.10.100'

gitlab-ctl reconfigure # 初始化配置
# 启动
gitlab-ctl start
gitlab-ctl status
gitlab-ctl stop
```

默认用户：root

```bash
# 获取默认密码
cat /etc/gitlab/initial_root_password
```

## docker

```bash
#搜索镜像
docker search gitlab
#下载镜像
sudo docker pull gitlab/gitlab-ce:latest

# 创建docker中的网络
docker network create gitlab_net
```

```bahs
docker run --name='gitlab-any' -d \
--net=gitlab_net \
--publish 443:443 --publish 80:80 --pulish 2222:22 \
--restart always \
--volume /root/gitlab/config:/etc/gitlab \
--volume /root/gitlab/logs:/var/log/gitlab \
--volume /root/gitlab/data:/var/opt/gitlab \
--privileged=true \
gitlab/gitlab-ce:15.6.2-ce.0
```

```bash
参数解析
1.http端口使用 80，ssh连接端口映射使用2222
2.网络使用 gitlab_net网络
3.将容器内部 /etc/gitlab,/var/log/gitlab,/var/opt/gitlab - 挂载到宿主机的 /root/docker/gitlab/config,logs,data 下，防止容器被删除数据丢失
4.privileged=true 使用特权，怕什么地方权限不足，安装不顺
5./root/docker/gitlab下的config,logs,data没有的话，创建容器会一并创建
```

修改配置文件

```bash
vim ~/docker/gitlab/config/gitlab.rb
...
#用户访问所使用的URL，域名或者IP地址
external_url 'http://gitlab.example.com/'
...
# 修改对应的ssh的连接端口也为映射的端口2222，不然无法在添加了ssh密钥之后，还是不能正常clone、pull、push代码
gitlab_rails['gitlab_shell_ssh_port'] = 2222
...
```

默认用户root

默认密码：

```bash
# 在挂载的目录下
cat root/docker/gitlab/config/initial_root_password
```

如果密码不对，重置密码：

```bash
# 进入 GitLab Rails Console
gitlab-rails console

# 在 GitLab Rails Console 中执行以下命令修改 root 用户密码：
user = User.find_by_username('root')
user.password = 'new_password'
user.password_confirmation = 'new_password'
user.save!

# 退出 GitLab Rails Console
exit
```

# gitlab-CI/CD

gitlab-CI是gitlab8.0之后自带的一个持续集成系统，中心思想是当每一次push到gitlab的时候，都会触发一次脚本执行，然后脚本内容包括了测试、编译、部署等一系列自定义内容

gitlab-CI的脚本执行，需要自定义安装对应gitlab-runner来执行，代码push之后，webhook检测到代码变化，就会触发gitlab-CI，分配到各个runner来运行相应的脚本script。这些脚本有的是测试项目用的，有的是部署用的。

![image-20240514165221598](./images/image-20240514165221598.png)

## 对比jenkins

**分支可配置性**

- 使用Gitlab-CI，新创建的分支无需任何一步配置即可立即使用CI管道中定义的作业
- Jenkina 2基本gitlab的多分支流水线可以实现，相对配置来说gitlab更加方便

**定时执行构建**

有时，根据时间触发作业或整个管道会有所帮助。例如，非常规时间，夜间定时构建

- 使用Jenkins 2可以立即使用，可以在应执行作业或管道中的那一刻以cron式语法定义
- Gitlab CI没有此功能。但是可以通过一种变通办法实现：通过webAPI使用同一台或另一台服务器上的cronjob触发作业或管道实现

**拉取请求支持**

如果很好地集成了存储库管理器和CI/CD平台，可以看到请求的当前构建状态，使用次功能，可以避免将代码合并到不起作用或无法正确构建的主分支中

- Jenkins没有与源代码管理系统进一步集成，需要管理员自行写代码或插件实现
- Gitlab与其CI平台紧密集成，可以方便查看每个打开或关闭拉取请求的运行和完成管道

**权限管理**

从存储库管理器的权限管理对于不想为每个服务分别设置每个用户的权限的大型开发人员或者组织团体很有用。大多数情况下，两种情况下的权限都是相同的，因此默认情况下应将他们配置在一个位置

- 由于Gitlab与GitlabCI的深度整合，权限可以统一管理
- 由于Jenkins 2没有内置的存储库管理器，因此它无法直接在存储库管理器和CI/CD平台之间合并权限

**存储库交互**

- Gitlab CI是Git存储库管理器Gitlab的固定组件，因此在CI/CD流程和存储库功能之间提供了良好的交互
- Jenkins 2与存储库管理器都是松散耦合的，因此在选择版本控制系统时非常的灵活。此外，Jenkins 2强调了对插件的支持，以进一步扩展或改善软件的现有功能

**插件管理**

- 扩展Jenkins的本机功能是通过插件完成的。插件的维护，保护和升级成本很高
- Gitlab是开放式的，任何人都可以直接向代码库贡献更改，一旦合并，他将自动测试并维护每个更改



GitlabCI

- 轻量级，不需要复杂的安装手段
- 配置简单，与gitlab可直接适配
- 实时构建日志十分清晰，UI交互体验很好
- 使用YAML进行配置，任何人都可以很方便的使用
- 没有统一的管理界面，无法统筹管理所有项目
- 配置依赖与代码仓库，耦合度没有Jenkins低
- 天然支持分布式，gitlab的runner可以装在任何一台电脑上，方便测试和集成

Jenkins

- 编译服务和代码仓库分离，耦合度低
- 插件丰富，支持语言众多
- 有统一的web管理界面
- 插件以及自身安装较为复杂
- 体量较大，不是很适合小型团队

GitLabCI有助于DevOps人员，例如敏捷开发中，开发和运维是同一个人，最便捷的开发方式。

JenkinsCI适合在多角色团队，职责分明、配置和代码分离、插件丰富

在使用过两者后，个人觉得gitlab-ci更简单易用，如果有gitlab-ci达不到的要求，可以考虑使用jenkins。

## 安装gitlab-runner

参考链接：https://docs.gitlab.com/runner/install/linux-manually.html

历史版本：https://gitlab.com/gitlab-org/gitlab-runner/-/tags

- 二进制

```bash

sudo curl -L --output https://gitlab.com/gitlab-org/gitlab-runner/-/releases/v15.6.1/downloads/binaries/gitlab-runner-linux-amd64
sudo chmod +x /usr/local/bin/gitlab-runner
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

查看状态

```bash
# gitlab-runner status
Runtime platform                                    arch=amd64 os=linux pid=65191 revision=133d7e76 version=15.6.1
```

设置Docker权限

让gitlab-runner能正确的执行docker命令，需要把gitlab-runner用户添加到docker group里，然后重启docker和gitlab ci runner

```bash
usermod -aG docker gitlab-runner
systemctl restart docker
gitlab-runner restart
```

- docker

```bash
docker run -d --name gitlab-runner --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
```



## gitlab-runner注册

注册流程是：获取runner token >> 注册

- **方法1：交互式注册**

![image-20240514173902408](./images/image-20240514173902408.png)

```bash
#  gitlab-runner register --url http://192.168.9.20:7780/ --registration-token 1xhn3LWAxuxVxd6QPhit
Runtime platform                                    arch=amd64 os=linux pid=67107 revision=133d7e76 version=15.6.1
WARNING: The 'register' command has been deprecated in GitLab Runner 15.6 and will be replaced with a 'deploy' command. For more information, see https://gitlab.com/gitlab-org/gitlab/-/issues/380872 
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
[http://192.168.9.20:7780/]: http://192.168.9.20:7780/
Enter the registration token:
[1xhn3LWAxuxVxd6QPhit]: 1xhn3LWAxuxVxd6QPhit
Enter a description for the runner:
[k8s-master1]: k8s-master1
Enter tags for the runner (comma-separated):

Enter optional maintenance note for the runner:

Registering runner... succeeded                     runner=1xhn3LWA
Enter an executor: virtualbox, docker-ssh+machine, docker, shell, parallels, ssh, docker+machine, instance, kubernetes, custom, docker-ssh:
virtualbox
Enter the VirtualBox VM (for example, my-vm):
any-vm
Enter the SSH user (for example, root):
any
Enter the SSH password (for example, docker.io):
123
Enter the path to the SSH identity file (for example, /home/user/.ssh/id_rsa):
/home/any/.ssh/id_rsa
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
 
Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml" 
```

```bash
● url：私有git的路径
● token：项目的token，用于关联runner和项目
● name：runner的名字，用于区分runner
● tags：用于匹配任务（jobs）和执行任务的设备（runners）
● executor：执行环境

其中url和token在项目的CI配置页上可以找到。name只是用来区分两个runner，没有特殊的作用。tags这个属性，job和runner都有，用来匹配任务和执行任务的runner。job的tags属性下一篇会提到，也可以自行查阅.gitlab-ci.yml的语法。runner的tag可以有多个，注册时用逗号（comma）分隔即可。当某个job的tag是当前runner tags的一个子集时，这个job就可以被分配到当前runner上执行。

举个例子：
runner的tag设为：python2.7,python3.4
job的tag设为：python2.7或python3.4，macos就可以在这个runner上执行。
job的tag设为：java，这个job就不会被分配到这个runner上。
```

executor就是执行job的环境，通常我们都会选择docker，如果有其他需要的也可以自行查阅文档。

当我们完成设置以后，可以通过`vi /etc/gitlab-runner/config.toml`打开runner的配置文件，刚才所填写的信息都会记录在其中。如果配置了多个runner，就会像图中一样，出现两个runners的section。

注册好之后

![image-20240514174940403](./images/image-20240514174940403.png)

- **方法2：直接注册**

```bash
gitlab-runner register \
   --non-interactive \
   --url "https://gitlab.example.com/" \
   --registration-token "AvpQDzBCL66sYKyURChH" \
   --executor "docker" \
   --description "runner" \
   --tag-list "run" \
   --run-untagged \
   --locked="false"

# 解释
--non-interactive：免交互式
--url：gitlab的连接地址
--registration-token gitlab的token
--executor：执行器
--description：runner名称
--tag-list：标签
--locked：是否开启锁定
```



gitlab-runner的类型

- shared ： 运行整个平台项目的作业（gitlab）
- group： 运行特定group下的所有项目的作业（group）
- specific: 运行指定的项目作业（project）
- locked： 无法运行项目作业
- paused： 不会运行作业

首先得知道gitlab-runner的类型有哪些，可以在不同的界面获取runner token就会生成不同类型的runner。

gitlab-runner是支持分布式的，可以运行在各种环境，极大的方便开发和测试，当安装好gitlab-runner之后，需要进行注册到gitlab上，进行关联，首先登陆gitlab获取url和tocken

获取shared类型runner token：

进入系统设置 -> Runners

1xhn3LWAxuxVxd6QPhit