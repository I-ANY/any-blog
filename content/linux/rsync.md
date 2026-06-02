+++
title = "rsync"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 3
+++

# 安装使用

```bash
yum -y install rsync
```

 两种传输方式：

1.pull 拉-> 下载 客户端将服务器上的数据下载到本地服务器 问题：如果客户端过多，会对服务端造成压力过大  

2.push 推-> 上传 客户端将本地数据上传到服务器 问题：如果客户端过多，上传速度会受到带宽影响，很慢



# 语法

## 本地模式语法

```bash
命令  选项    源数据    目标地址 
rsync [OPTION]... SRC [SRC]... DEST  

常用选项：rsync支持一百多个选项，所以此处只介绍几个常用选项
-a --archive ：归档模式，表示递归传输并保持文件属性。等同于"-rtopgDl"。 
-v：显示rsync过程中详细信息。可以使用"-vvvv"获取更详细信息。 
-z：传输时进行压缩提高效率。 
-R：使用相对路径。


其他选项：
-n ：如果不确定 rsync 执行后会产生什么结果，可以先用`-n`或`--dry-run`参数模拟执行的结果。
rsync -anv source/ destination
 
--delete :如果要使得目标目录成为源目录的镜像副本，将删除只存在于目标目录、不存在于源目录的文件，也就是"多则删之，少则补之"。
rsync -az --delete /aaa/ /bbb/
 
--exclude:排除文件或者目录，不进行拷贝（注意，rsync 会同步以"点"开头的隐藏文件，如果要排除隐藏文件，可以这样写--exclude=".*"。）
rsync -av --exclude='*.txt' source/ destination
 
--include：必须同步的文件，通常来配合--exclude使用
rsync -av --include="[0-9].txt" --exclude='*' source/ destination
```

 

类似于cp命令，又不同于cp 

（1）、cp命令只是本地复制，每次cp都会用源文件内容覆盖新文件,所以cp命令会修改文件时间属性； 

（2）、rsync可本地可远程，首次rsync与cp一样，后续rsync会对比两个文件的不同，只传输文件更新的部分,如果未更新，则rsync不会修改文件任何属性  注意：源路径如果是一个目录的话，带上尾随斜线和不带尾随斜线是不一样的，不带尾随斜线表示的是整个目录包括目录本身，带上尾随斜线表示的是目录中的文件，不包括目录本身，这一点本地模式与远程模式均适用

```bash
# 示例1：
# 本机拷贝文件（rsync首次拷贝为全量，后续拷贝如果参照的源内容不变，则不覆盖目标文件)
rsync /etc/passwd /test

# 本机拷贝目录：
rsync -r /etc /tmp
 
# 示例2：
# --backup:对目标目录下已经存在的文件做一个备份
rsync -r --backup /test111/ /test222/
```

## 远程模式

**1、ssh协议**

（1）在本地与目标主机都安装rsync 

（2）远程主机要打开sshd服务 

（3）需要用到的账号是远程主机可登录系统账号---》不安全 

（4）不受目录限制-------------------------》不安全

**2、rsync协议**

（1）在本地与目标主机都安装rsync 

（2）远程主机要打开rsync守护进程 ：rsync --daemon   或   systemctl start rsyncd
（3）用到的是虚拟账号，虚拟账号对应的是配置文件中uid的权限，例如：uid=rsync   虚拟账号any------->远端主机真实存在的系统账号rsync 

（4）用的是模块名-》具体的目录



**语法**

语法类似于scp命令，但备份方案不同于scp（scp 与 cp一样，每次都是全量)

```bash


# 拉取 
命令     选项    远程用户@远程主机:远程的目录作为源  本地作为目标 
rsync [OPTION]  [USER@]HOST:SRC                [DEST]  

# 推送 
命令   选项      本地数据作为源          远程用户@远程主机:远程的目录作为目标 
rsync [OPTION]  SRC [SRC]...          [USER@]HOST:DEST
 
# 示例：拉取
rsync -avz root@192.168.12.17:/data /bak/

# 示例：推送
rsync -avz /bak root@192.168.12.17:/data/
```



# 实时同步：+inotify

通过rsync+inotify组合可以实现实时同步，制作异地镜像站点，以便为异地备份做好准备工作

```bash
# 安装inotify工具：inotify-tools
# yum安装：依赖epel源
yum install -y epel-release inotify-tools

```



提供了两个命令：

- **inotifywait**

```bash
等待文件变化
参数：
	• -m, --monitor：持续监控模式，保持监听状态，不退出。
	• -r, --recursive：递归监控指定目录及其子目录。
	• -q, --quiet：安静模式，不输出事件信息。
	• -t <seconds>, --timeout <seconds>：设置超时时间，超过指定时间后退出程序。
	• -e，--event <event>：指定要监控的事件，多个事件可以用逗号分隔。常见事件包括：
		create：在被监控的目录中创建了文件或目录
    delete：删除了被监控目录中的某文件或目录
    modify：修改，文件内容被修改
    access：文件被访问
    attrib：元数据被修改。包括权限、时间戳、扩展属性等等

    close_write：打开的文件被关闭，是为了写文件而打开文件、随后被关闭的事件，比如用vim编辑文件或者echo 1111 >> egon.txt 
    close_nowrite：read only模式下文件被关闭，即只能是为了读取而打开文件，读取结束后关闭文件的事件
    close：是close_write和close_nowrite的结合，无论是何种方式打开文件，只要关闭都属于该事件
    open：文件被打开
    moved_to：向监控目录下移入了文件或目录，也可以是监控目录内部的移动
    moved_from：将监控目录下文件或目录移动到其他地方，也可以是在监控目录内部的移动
    move：是moved_to和moved_from的结合
    moved_self：被监控的文件或目录发生了移动，移动结束后将不再监控此文件或目录
    delete_self：被监控的文件或目录被删除，删除之后不再监控此文件或目录
    umount：挂载在被监控目录上的文件系统被umount，umount后不再监控此目录
    isdir ：监控目录相关操作
		
--format:指定输出的格式
	• %w：被监控的文件或目录的路径。
	• %f：事件发生的文件或目录的名称。
	• %e：触发事件的类型。
	• %T：事件发生的时间戳。
	• %:e：根据事件类型返回对应的文本描述（例如，CREATE、MODIFY等）。


```

- **inotifywatch**

收集文件系统统计的数据，例如发生了多少次inotify事件，某文件被访问了多少次等等

rsync+inotify脚本，该脚本需要运行在本地 ，并且需要远端开启rsyncd

```bash
 
#!/bin/bash  
watch_dir=/test/        # 本地被监控目录 
user="any"          # 虚拟用户 
export RSYNC_PASSWORD=123   # 虚拟用户密码 
module="xxx"          # 远程模块名 
ip=192.168.12.39        # 远程主机ip  

# 先整体同步一次 
rsync -azc --delete ${watch_dir} ${user}@${ip}::${module}  

# 切换到被监控目录下，然后用inotifywait监控./目录，这样后期就可以用-R选项同步新增的子目录 
cd $watch_dir  /usr/bin/inotifywait -mrq --timefmt '%Y-%m-%d %H:%M:%S' --format '%w%f:%Xe:%T' -e create,delete,modify,move,attrib,close_write ./ \ --exclude=".*.swp" | \
while read line do   
	# $line的输出format为：文件路径:事件:时间   
	FILE=$(echo $line | awk -F: '{print $1}') # 获取文件的绝对路径   
	EVENT=$(echo $line | awk -F: '{print $2}') # 获取监控的事件    
	
	# 监控到对文件的下述行为后，只把文件同步到远端   
	if [[ $EVENT =~ 'CREATE' ]] || [[ $EVENT =~ 'MODIFY' ]] || [[ $EVENT =~ 'CLOSE_WRITE' ]] || [[ $EVENT =~ 'MOVED_TO' ]] || [[ $EVENT =~ 'ATTRIB' ]];then     
	rsync -azcR ${FILE} ${user}@${ip}::${module}   
	fi      
	
	# 监控到涉及到目录的改动，将目录同步到远端，例如用dirname ${FILE}获取目录   
	if [[ $EVENT =~ 'DELETE' ]] || [[ $EVENT =~ 'MOVED_FROM' ]];then     
	rsync -azcR --delete $(dirname ${FILE}) ${user}@${ip}::${module} &>/dev/null   
	fi done & 
	# 末尾的&符号，代表在子shell中提交命令，这样进程的ppid就变为1，当前窗口关闭，该进程依然存活
```