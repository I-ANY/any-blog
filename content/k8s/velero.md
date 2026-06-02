+++
title = "velero"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 15
+++

# 简介

参考博客：https://www.cnblogs.com/nf01/p/18118159

Velero 是一款云原生时代的灾难恢复和迁移工具，采用 Go 语言编写，并在 github 上进行了开源，利用 velero 用户可以安全的备份、恢复和迁移 Kubernetes 集群资源和持久卷。

- 开源地址：https://github.com/vmware-tanzu/velero
- 官方文档：https://velero.io/docs/v1.11/

![image-20240430173735648](./images/image-20240430173735648.png)

版本支持列表：

<img src="./images/image-20240430162348740.png" alt="image-20240430162348740" style="zoom:67%;" />

Velero 组件一共分两部分，分别是服务端和客户端。

- **服务端**：运行在你 Kubernetes 的集群中
- **客户端**：是一些运行在本地的命令行的工具，需要已配置好 kubectl 及集群 kubeconfig 的机器上

**velero备份流程：**

1. velero客户端调用kubernetes API Server创建backup任务
2. Backup控制器基于watch机制通过Api Server获取到备份任务
3. Backup控制器开始执行备份动作，会通过请求Api Server获取到需要备份的数据
4. Backup 控制器将获取到的数据备份到指定的对象存储server端

![image-20240430173757657](./images/image-20240430173757657.png)

**Velero后端存储：**

`Velero`支持两种关于后端存储的`CRD`，分别是`BackupStorageLocation`和`VolumeSnapshotLocation`。

-  BackupStorageLocation

​	主要用来定义 Kubernetes 集群资源的数据存放位置，也就是集群对象数据，不是 PVC 的数据。主要支持的后端存储是 S3 兼容的存储，比如：Mino 和阿里云 OSS 等。

- VolumeSnapshotLocation

​	主要用来给 PV 做快照，需要云提供商提供插件。阿里云已经提供了插件，这个需要使用 CSI 等存储机制。你也可以使用专门的备份工具 `Restic`，把 PV 数据备份到阿里云 OSS 中去(安装时需要自定义选项)。

# 安装

## minio客户端

在 [Github Release 页面](https://github.com/vmware-tanzu/velero/releases)下载指定的 velero 二进制客户端安装包，比如这里我们下载我们k8s集群对应的版本为 `v1.11.1`

版本列表：https://github.com/vmware-tanzu/velero/releases

```bash
$ wget https://github.com/vmware-tanzu/velero/releases/download/v1.11.1/velero-v1.11.1-linux-amd64.tar.gz
$ tar zxf velero-v1.11.1-linux-amd64.tar.gz
$ mv velero-v1.11.1-linux-amd64/velero /usr/bin/
$ velero -h
# 启用命令补全
$ source <(velero completion bash)
$ velero completion bash > /etc/bash_completion.d/velero

```

## minio服务端

使用minio作为后端存储（安装可参考minio篇）

Velero支持很多种存储插件，可查看：[Velero Docs - Providers](https://velero.io/docs/v1.13/supported-providers/)获取插件信息，这里使用minio作为S3兼容的对象存储提供程序。也可以在任意地方部署Minio对象存储，只需要保证K8S集群可以访问到即可。

可以使用 velero 客户端来安装服务端，也可以使用 Helm Chart 来进行安装，这里用客户端来安装，velero 命令默认读取 kubectl 配置的集群上下文，所以前提是 velero 客户端所在的节点有可访问集群的 kubeconfig 配置。

### 创建密钥

首先准备密钥文件，在当前目录建立一个空白文本文件，内容如下所示：

```bash
$ cat > credentials-velero <<EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
```

### [安装velero到k8s集群](https://github.com/vmware-tanzu/velero-plugin-for-aws#setup)

替换为之前步骤中 minio 的对应 access key id 和 secret access key如果 minio 安装在 kubernetes 集群内时按照如下命令安装 velero 服务端:

```bash
$ velero install \
  --provider aws \
  --image velero/velero:v1.11.1 \
  --plugins velero/velero-plugin-for-aws:v1.7.1 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --use-volume-snapshots=false \
  --namespace velero \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio:9000 \
  --wait
# 执行install命令后会创建一系列清单，包括CustomResourceDefinition、Namespace、Deployment等。
# velero install .... --dry-run -o yaml > velero_deploy.yaml 如果为私仓，可以通过--dry-run 导出 YAML 文件调整在应用。

# 可使用如下命令查看运行日志
$ kubectl logs deployment/velero -n velero
 
# 查看velero创建的api对象
$ kubectl api-versions | grep velero
velero.io/v1
 
# 查看备份位置
$ velero backup-location get
NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   aws        velero          Available   2023-03-28 15:45:30 +0800 CST   ReadWrite     true

#卸载命令
# velero uninstall --namespace velero
You are about to uninstall Velero.
Are you sure you want to continue (Y/N)? y
Waiting for velero namespace "velero" to be deleted
............................................................................................................................................................................................
Velero namespace "velero" deleted
Velero uninstalled ⛵
```

参数说明

```bash
--kubeconfig(可选)：指定kubeconfig认证文件，默认使用.kube/config；
--provider：定义插件提供方；
--image：定义运行velero的镜像，默认与velero客户端一致；
--plugins：指定使用aws s3兼容的插件镜像；
--bucket：指定对象存储Bucket桶名称；
--secret-file：指定对象存储认证文件；
--use-node-agent：创建Velero Node Agent守护进程，托管FSB模块；
--use-volume-snapshots：是否启使用快照；
--namespace：指定部署的namespace名称，默认为velero；
--backup-location-config：指定对象存储地址信息；
```

aws插件与velero版本对应关系：

https://github.com/vmware-tanzu/velero-plugin-for-aws

<img src="./images/image-20240430163104534.png" alt="image-20240430163104534" style="zoom:50%;" />

### 卸载velero

如果您想从集群中完全卸载Velero，则以下命令将删除由`velero install`创建的所有资源:

```bash
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero

```

# 备份

备份命令：

```bash
velero create backup NAME [flags]
```

**backup选项：**

- `--exclude-namespaces stringArray` : 要从备份中排除的名称空间
- `--exclude-resources stringArray`: 要从备份中排除的资源，如`storageclasses.storage.k8s.io`
- `--include-cluster-resources optionalBool[=true]`: 包含集群资源类型
- `--include-namespaces stringArray`: 要包含在备份中的名称空间(默认'*')
- `--include-resources stringArray`: 备份中要包括的资源
- `--labels mapStringString`: 给这个备份加上标签
- `-o, --output string`: 指定输出格式，支持'table'、'json'和'yaml'；
- `-l, --selector labelSelector`: 对指定标签的资源进行备份
- `--snapshot-volumes optionalBool[=true]`: 对 PV 创建快照
- `--storage-location string`: 指定备份的位置
- `--ttl duration`: 备份数据多久删掉
- `--volume-snapshot-locations strings`: 指定快照的位置，也就是哪一个公有云驱动



## 示例测试

```bash
# 备份命名空间
velero backup create nginx-backup --include-namespaces nginx-example

# 参数：
# --include-namespaces：指定命名空间
# --selector：标签选择器，如app=nginx
# --default-volumes-to-fs-backup：默认将所有PV卷进行备份，详情查看官方文档：https://velero.io/docs/v1.10/file-system-backup/
```

查询备份列表

```bash
velero backup get
 
 # 查看备份详细信息
velero backup describe nginx-backup
 
# 查看备份日志
velero backup logs nginx-backup
```

## 定时备份

使用cron表达式备份

```bash
$ velero schedule create nginx-daily --schedule="0 1 * * *" --include-namespaces nginx-example
```

使用一些非标准的速记 cron 表达式

```bash
$ velero schedule create nginx-daily --schedule="@daily" --include-namespaces nginx-example
```

手动触发定时任务

```bash
$ velero backup create --from-schedule nginx-daily
```

更多cron示例参考：[cron package’s documentation](https://pkg.go.dev/github.com/robfig/cron#hdr-Predefined_schedules)

批量备份全部命名空间

```bash
cat >all-ns-backup.sh <<EOF
#!/bin/bash
NS_NAME=`kubectl get ns | awk '{if (NR>2){print}}' | awk '{print $1}'`
DATE=`date +%Y%m%d%H%M%S`
cd /data/velero/
for i in $NS_NAME;do
velero backup create ${i}-ns-backup-${DATE} \
--include-cluster-resources=true \
--include-namespaces ${i} \
--kubeconfig=/root/.kube/config \
--namespace velero-system
done
EOF

root@k8s-master01:/data/velero# 

```



# 恢复

模拟故障

```bash
# 删除nginx-example命名空间和资源
$ kubectl delete namespace nginx-example
# 检查是否删除
$ kubectl get all -n nginx-example
No resources found in nginx-example namespace.
```

恢复资源

```bash
$ velero backup get
NAME           STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
nginx-backup   Completed   0        0          2024-04-06 21:47:16 +0800 CST   29d       default            <none>

$ velero restore create --from-backup nginx-backup
Restore request "nginx-backup-20240406215611" submitted successfully.
Run `velero restore describe nginx-backup-20240406215611` or `velero restore logs nginx-backup-20240406215611` for more details.

```

检查

```bash
# 查看恢复的资源
$ velero restore get
 
# 查看详细信息
$ velero restore describe nginx-backup-20240406215611
 
# 检查资源状态
$ kubectl get all -n nginx-example

```



# 项目迁移

## 项目介绍

我们将使用`Velero`快速完成**云原生应用**及**PV数据**的迁移实践过程，在本文示例中，我们将**A集群**中的一个**MOS应用**迁移到**集群B**中，其中数据备份采用自建**Minio**对象存储服务。

**环境要求:** 

1. 迁移项目最好保证两个Kubernetes集群版本一致。
2. 为了保证PV数据成功迁移，两边需要安装好相同名字的`StorageClass`。
3. 可以自己部署Minio，也可以使用公有云的对象存储服务，如华为的OBS、阿里的OSS等。
4. 本案例将集群A中app-system命名空间中的服务及PV数据迁移到集群B中。

**项目说明**

需要将集群A中 `istio-system` 空间的所有资源和数据全部迁移到集群B中，该项目包括了`deployment`、`statefulset`、`service`、`ingress`、`job`、`cronjob`、`secret`、`configmap`、`pv`、`pvc`。











