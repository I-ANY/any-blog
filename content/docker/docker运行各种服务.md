+++
title = "docker运行各种服务"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 2
+++

# docker运行mysql

1、获取mysql镜像

```bash
docker search mysql

# 最新版的MySQL
docker pull mysql:last
# 指定版的MySQL
docker pull mysql:5.6.x
docker pull mysql:5.7.x

```

2、创建本地目录

需要将MySQL的数据持久化到宿主机，也就是建立映射，包括配置文件、数据文件和log日志目录

```bash
# 把上述三个目录创建好
mkdir -p /docker_data/mysql_data/data /docker_data/mysql_data/logs /docker_data/mysql_data/conf
# 创建一个cnf文件
touch /docker_data/mysql_data/conf/my.cnf
```

3、启动mysql容器

```bash
docker run -p 3307:3306 --name mysql --restart=always -v /docker_data/mysql_data/conf:/etc/mysql/conf.d -v /docker_data/mysql_data/logs:/logs -v   /docker_data/mysql_data/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=1234 -d mysql:5.7.29

# 参数
-p 3306:3306：将mysql的3306映射到宿主机的3306端口。
--name mysql：为容器起个名称。
-v：建立容器与宿主机的目录映射。
-e：添加环境变量，这里是为root用户设置密码。
-d：后台运行容器
mysql:5.7.29：这个mysql容器基于mysql:5.7.29镜像。
```



# docker运行redis

1、拉取redis镜像

```bash
# 查询可用镜像
docker search redis

# 拉取镜像, 下面命令是最新版
docker pull redis

# 查看拉取的镜像文件
docker images | grep redis
```

2、启动一个redis容器

```bash
# 创建本地持久化目录
[root@r ~]# mkdir -p /docker_data/redis_data/

# 启动
docker run --name myredis -p 6379:6379 --restart=always -v /docker_data/redis_data/data:/data -d redis redis-server --appendonly yes

[root@r ~]# docker run --name myredis -p 6379:6379 --restart=always -v /docker_data/redis_data:/data -d redis redis-server --appendonly yes
7c322e3633d2dfa41e0cf79a5b3aa748ccc7f18ef44c82caec6e92a46e55f8d9

[root@r ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
7c322e3633d2        redis               "docker-entrypoint.s…"   3 seconds ago       Up 3 seconds        0.0.0.0:6379->6379/tcp              myredis

```

参数说明

参数说明：

- `--name myredis`，指定该容器名称，查看和进行操作都比较方便
- `-p 6379:6379`，将redis的`6379`端口(冒号右边)映射到宿主机的`6379`端口(冒号左边)
- `-v /docker_data/redis_data:/data`,文件挂载
- `-d redis redis-server`，后台启动
- `--appendonly yes`，开启redis 持久化

测试

```bash
docker exec -it 容器ID bash

[root@r ~]# docker exec -it 7c322e3633d2 bash
root@7c322e3633d2:/data# redis-cli
127.0.0.1:6379> set name a
OK
127.0.0.1:6379> get name
"a"
127.0.0.1:6379> exit
root@7c322e3633d2:/data# exit

```



# docker运行tomcat部署java项目

**1、拉取tomcat镜像**

```bash
# 查找镜像
docker search tomcat
#结果会有很多的Tomcat镜像，通常情况下，我们都知道官方的东西基本上代表安全无公害，因为可以看到右边有official标识为OK的就是官方的镜像，因此我们拉取第一个Tomcat就可以。

# 拉取镜像
docker pull tomcat
docker pull tomcat:TAG
```



**2、添加java的web应用**
首先，将本地的java war包，先上传到服务器的指定目录。

然后将该目录挂载到Tomcat的webapps目录内，并且启动

```bash
[root@r zhangkai]# docker run -d --restart=always -v /home/java_web/pinter.war:/usr/local/tomcat/webapps/pinter.war -p 6002:8080 docker.io/tomcat
# 将/home/java_web/目录中的pinter.war包挂在到tomcat/usr/local/tomcat/webapps/目录中，并命名为pinter.war，然后宿主机的端口是6002与Tomcat镜像监听的8080端口，docker.io/tomcat是Tomcat的镜像名。
```



3、浏览器访问主机的6002端口即可







