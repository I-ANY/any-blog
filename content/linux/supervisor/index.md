+++
title = "supervisor"
date = "2026-05-28T00:01:08+08:00"
draft = false
+++

# 介绍

Supervisor ([http://supervisord.org](https://link.jianshu.com/?t=http://supervisord.org/)) 是一个用 [Python] 写的进程管理工具，可以很方便的用来启动、重启、关闭进程(不仅仅是 Python 进程)。除了对单个进程的控制，还可以同时启动、关闭多个进程，比如很不幸的服务器出问题导致所有应用程序都被杀死，此时可以用 supervisor 同时启动所有应用程序而不是一个一个地敲命令启动

# 安装

- 在线安装

```bash
# 可以通过apt-get、yum安装，既然Supervisor是基于python编写的，那我们就用pip安装好了

# 一、在线安装
# 1 yum安装，配置好yum源后，可以直接安装
yum install supervisor
# Debian/Ubuntu可通过apt安装
apt-get install supervisor

# 2 pip安装
yum install python-setuptools-devel
pip install supervisor

# 3 easy_install安装
yum install python-setuptools-devel
easy_install supervisor
```



- 离线安装

1、setuptools安装

安装包地址：https://pypi.org/project/setuptools/#files

```bash
tar -zxvf setuptools-0.6c11.tar.gz 
cd setuptools-0.6c11 
python setup.py install
```

2、meld3安装

安装包地址：https://pypi.org/project/meld3/#files

```bash
tar -zxvf meld3-1.0.2.tar.gz 
cd meld3-1.0.2 
python setup.py install
```

3、supervisor安装

安装包地址：https://pypi.org/project/supervisor/#files

```bash
tar -zxvf supervisor-3.3.1.tar.gz 
cd supervisor-3.3.1 
python setup.py install
```



# 配置

## 主配置文件

```bash
# 在线安装完成后，会在 /etc 下创建一个 supervisord.d目录用于存放supervisor的配置文件,还有一个supervisord.conf配置文件
# 如果没有使用命令生成：（离线安装直接命令生成） 
echo_supervisord_conf > /etc/supervisord.conf

# 一般把supervisor服务器相关的配置写入supervisord.conf中,把监控各个进程的配置，按照进程名存在 supervisord.conf 目录下。（这个可以在supervisord.conf中的[include]部分下配置）
```

**supervisord.conf** 配置文件介绍

```bash
# 简单说明:
[unix_http_server]  #配置socket连接部分
[supervisord] 			#配置supervisor服务器部分
[supervisorctl] 		#配置supervisor客户端部分
[inet_http_server] 	#配置web管理界面
[include] 					#配置需要引入的其他配置

# 详细介绍：
[unix_http_server]
file=/tmp/supervisor.sock   ; UNIX socket 文件，supervisorctl 会使用
;chmod=0700                 ; socket 文件的 mode，默认是 0700
;chown=nobody:nogroup       ; socket 文件的 owner，格式： uid:gid

;[inet_http_server]         ; HTTP 服务器，提供 web 管理界面
;port=127.0.0.1:9001        ; Web 管理后台运行的 IP 和端口，如果开放到公网，需要注意安全性
;username=user              ; 登录管理后台的用户名
;password=123               ; 登录管理后台的密码

[supervisord]
logfile=/tmp/supervisord.log ; 日志文件，默认是 $CWD/supervisord.log
logfile_maxbytes=50MB        ; 日志文件大小，超出会 rotate，默认 50MB
logfile_backups=10           ; 日志文件保留备份数量默认 10
loglevel=info                ; 日志级别，默认 info，其它: debug,warn,trace
pidfile=/tmp/supervisord.pid ; pid 文件
nodaemon=false               ; 是否在前台启动，默认是 false，即以 daemon 的方式启动
minfds=1024                  ; 可以打开的文件描述符的最小值，默认 1024
minprocs=200                 ; 可以打开的进程数的最小值，默认 200

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; 通过 UNIX socket 连接 supervisord，路径与 unix_http_server 部分的 file 一致
;serverurl=http://127.0.0.1:9001 ; 通过 HTTP 的方式连接 supervisord

; 包含其他的配置文件
[include]
files = relative/directory/*.ini    ; 可以是 *.conf 或 *.ini



# 修改配置文件
vim supervisord.conf 
# 最后一行改为（;表示注释），这样配置文件可以写到supervisord.d目录下一xx.ini命名，xx.conf文件也行
[include]
files = supervisord.d/*.ini

```

## 子配置文件

```bash
# 在/etc/supervisord.d/目录下所有 *.ini或者*.conf都会被管理
vi /etc/supervisord.d/redis.ini

[program:redis-server]
command=/usr/bin/redis-server /etc/redis/6379.conf   ; 需要执行的命令
priority=999                ; 优先级（越小越优先）
autostart=true              ; supervisord启动时，该程序也启动
autorestart=true            ; 异常退出时，自动启动
startsecs=10                ; 启动后持续10s后未发生异常，才表示启动成功
startretries=3              ; 异常后，自动重启次数
exitcodes=0,2               ; exit异常抛出的是0、2时才认为是异常
stopsignal=QUIT             ; 杀进程的信号

; 在程序发送stopignal后，等待操作系统将SIGCHLD返回给supervisord的秒数。
; 如果在supervisord从进程接收到SIGCHLD之前经过了这个秒数，
; supervisord将尝试用最终的SIGKILL杀死它
stopwaitsecs=1
user=root                   ; 设置启动该程序的用户
log_stdout=true             ; 如果为True，则记录程序日志
log_stderr=false            ; 如果为True，则记录程序错误日志
logfile=/var/log/redis-server.log    ; 程序日志路径
logfile_maxbytes=1MB        ; 日志文件最大大小
logfile_backups=10          ; 日志文件最大数量
```

# 启停

```bash
# yum安装的话直接systemctl启停
systemctl start supervisord
systemctl status supervisord
#5 开机自启
systemctl enable supervisord


# 其他方式安装的话
# 1 启动supervisord
supervisord -c /etc/supervisord.conf   或  supervisord 

# 2 停止supervisord
supervisorctl shutdown

# 3 重新加载配置文件
supervisorctl reload

# 4 注意：如果配置了密码（使用如下命令）
supervisorctl -u user -p 123 reload

```

可自己配置systemctl管理服务

```bash
# 1 新建配置文件
vim  /usr/lib/systemd/system/supervisord.service
```

配置内容如下

```bash
[Unit]
Description=supervisor
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/supervisord -c /etc/supervisord.conf
ExecStop=/usr/local/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/local/bin/supervisorctl $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

```bash
# 重新加载服务
systemctl daemon-reload

# 启动
systemctl start supervisord
systemctl status supervisord
# 开机自启
systemctl enable supervisord
```

# 命令行进程管理

```bash
# 更新配置
supervisorctl update
# 重新启动配置中的所有程序
supervisorctl reload

# 启动supervisord管理的所有进程
supervisorctl start all
# 停止supervisord管理的所有进程
supervisorctl stop all
# 启动supervisord管理的某一个特定进程
supervisorctl start program-name   # program-name为[program:xx]中的xx
# 停止supervisord管理的某一个特定进程
supervisorctl stop program-name  // program-name为[program:xx]中的xx

# 重启所有进程或所有进程
supervisorctl restart all 
# 重启所有supervisorctl reatart program-name 
# 重启某一进程，program-name为[program:xx]中的xx
# 查看supervisord当前管理的所有进程的状态
supervisorctl status
```

```bash
# 进入 supervisorctl 的 shell界面，执行命令：
supervisorctl -c /etc/supervisord.conf
# 或者启动后，直接使用 （进入shell界面）
supervisorctl 

> status    # 查看程序状态
> stop program-name    # 关闭 program-name  程序
> start program-name   # 启动 program-name  程序
> restart program-name     # 重启 program-name  程序
> reread    # 读取有更新（增加）的配置文件，不会启动新添加的程序
> update    # 重启配置文件修改过的程序
```



# web页面管理

```bash
# 1 修改配置文件
vim /etc/supervisord.conf 
# 2 修改内容如下
[inet_http_server]         ; inet (TCP) server disabled by default
port=0.0.0.0:9001        ; (ip_address:port specifier, *:port for all iface)
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

# 3 重启
supervisorctl reload
# 4 在浏览器打开：http://本机ip:8080/可以看到
```

