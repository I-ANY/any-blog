+++
title = "kustormize"
date = "2026-05-28T00:01:08+08:00"
draft = false
+++

# 简介

官网：https://kubectl.docs.kubernetes.io/

一般应用都会存在多套部署环境：开发环境、测试环境、生产环境，多套环境意味着存在多套 K8S 应用资源 YAML。而这么多套 YAML 之间只存在微小配置差异，比如镜像版本不同、Label 不同等，而这些不同环境下的YAML 经常会因为人为疏忽导致配置错误。再者，多套环境的 YAML 维护通常是通过把一个环境下的 YAML 拷贝出来然后对差异的地方进行修改。一些类似 Helm 等应用管理工具需要额外学习DSL 语法。总结以上，在 k8s 环境下存在多套环境的应用，经常遇到以下几个问题：

- 如何管理不同环境或不同团队的应用的 Kubernetes YAML 资源
- 如何以某种方式管理不同环境的微小差异，使得资源配置可以复用，减少 copy and change 的工作量
- 如何简化维护应用的流程，不需要额外学习模板语法

Kustomize 通过以下几种方式解决了上述问题：

- kustomize 通过 Base & Overlays 方式(下文会说明)方式维护不同环境的应用配置
- kustomize 使用 patch 方式复用 Base 配置，并在 Overlay 描述与 Base 应用配置的差异部分来实现资源复用
- kustomize 管理的都是 Kubernetes 原生 YAML 文件，不需要学习额外的 DSL 语法

# kustomize概念

- **kustomization**

​	指的是 kustomization.yaml 文件，或者指的是包含 kustomization.yaml 文件的目录以及它里面引用的所有相关文件路径，用于声明基本资源，以及如何修改这些资源。

- **base**

​	一个目录，包含该应用基本 的Kubernetes 资源配置文件及对应的kustomization ，任何的 kustomization 包括 overlay (后面提到)，都可以作为另一个 kustomization 的 base (简单理解为基础资源)。base 中描述了原始的 Kubernetes YAML 文件，如资源和常见的资源配置

- **overlay**

​	overlay 是一个目录，包含用于不同环境（如预生产、生产和两个测试环境）的覆盖层配置。每个子目录（如`pre`、`prod`、`test1` 和 `test2`）都有自己的kustomization，代表一个环境，包含针对该环境的特定修改。它修改(并因此依赖于)另外一个 kustomization，overlay 中的 kustomization指的是一些其它的 kustomization（也就是 base，没有 base，overlay 无法使用），并且一个 overlay 可以用作另一个 overlay 的 base(基础)。简而言之，overlay 声明了与 base 之间的差异。通过 overlay 来维护基于 base 的不同 variants(变体)，例如开发、QA 和生产环境的不同 variants

- **variant**

​	variant 是在集群中将 overlay 应用于 base 的结果。例如开发和生产环境都修改了一些共同 base 以创建不同的 variant。这些 variant 使用相同的总体资源，并与简单的方式变化，例如 deployment 的副本数、ConfigMap使用的数据源等。简而言之，variant 是含有同一组 base 的不同 kustomization

- **resource**

​	在 kustomize 的上下文中，resource 是描述 k8s API 对象的 YAML 或 JSON 文件的相对路径。即是指向一个声明了 kubernetes API 对象的 YAML 文件

- **patch**

​	修改文件的一般说明。文件路径，指向一个声明了 kubernetes API patch 的 YAML 文件

目录结构示例：

```bash
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── kustomization.yaml
    │   └── patch.yaml
    ├── prod
    │   ├── kustomization.yaml
    │   └── patch.yaml
    └── staging
        ├── kustomization.yaml
        └── patch.yaml
```



# 客户端安装

官网：https://kubectl.docs.kubernetes.io/installation/kustomize/

在kubernetes 1.14版本以上，已经集成到kubectl中了，你可以通过kubectl --help来进行查看命令。

```yaml
[root@k8s-master ~]# kubectl kustomize -h 
kustomize     Build a kustomization target from a directory or URL.
```

但是内置的kustomize功能较少，推荐安装完整的kustomize客户端，使用更多功能。

```bash
# wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.0.1/kustomize_v5.0.1_linux_amd64.tar.gz
# tar -zxvf kustomize_v5.0.1_linux_amd64.tar.gz -C /usr/local/bin/
# kustomize version
v5.0.1
```

常用命令

- kubectl

```bash
# 不创建资源，预览资源文件，也不会覆盖原来文件，只是输出到前端
kubectl kustomize ./
# 客户端命令
kustomize 

# 创建资源
kubectl apply -k ./

# 查看资源对象
kubectl get -k ./ 

# 查看资源详情
kubectl describe -k ./

# 比较应用前后资源在集群将处于的状态
kubectl diff -k ./ 

# 删除资源
kubectl delete -k ./
```

- Kustomize客户端

```bash
# 于生成最终应用于 Kubernetes 集群的 YAML 配置，生成的结果可以通过 kubectl apply -f 等命令应用于集群。
kustomize build

# 该命令允许用户直接在命令行中修改 Kustomization 中的镜像及其标签
# 例如: kustomize edit set image busybox=alpine:3.6 会将 busybox 镜像修改为 alpine:3.63。
kustomize edit set image

# 创建新的 Kustomization 文件夹和基本的 kustomization.yaml 文件
kustomize create
```



# 常用方法

以下全局基本的deployment资源文件

```yaml
cat > demo.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        ports:
        - containerPort: 80
        volumeMounts:
        - name: my-conf
          mountPath: /etc/conf 
      volumes:
      - name: my-conf
        configMap:
          name: my-configMap
EOF

```



## configMapGenerator

### 基于文件生成 ConfigMap

```bash
# 准备一个.properties文件
cat >application.properties <<EOF
FOO=Bar
EOF

# 准备kustomization.yaml文件
cat >kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1   # 接口，可以省略不写
kind: Kustomization   # 资源类型，可以省略不写

resources:
  - demo.yaml         # 修改的目标资源文件

configMapGenerator:   # 使用configMapGenerator生成配置
- files:
  - application.properties           # 原配置文件名，需跟当前文件同级
  name: my-configMap  # cm名称
EOF

```

所生成的configMap资源文件

```yaml
apiVersion: v1
data:
  application.properties: |
    FOO=Bar
kind: ConfigMap
metadata:
  name: my-configMap-g4hk9g2ff8
```

### 基于env 文件生成 ConfigMap

```bash
# 创建一个 .env 文件
cat >.env <<EOF
FOO=Bar
EOF

# 准备kustomization.yaml文件
cat >./kustomization.yaml <<EOF
# 省略apiVersion与kind字段不写
resources:
  - demo.yaml         # 修改的目标资源文件

configMapGenerator:
- name: example-configmap
  envs:
  - .env
EOF

```

所生成的configMap资源文件

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  name: example-configmap-42cfbf598f
```

### 基于字面键值对生成

```bash
cat >./kustomization.yaml <<EOF
resources:
  - demo.yaml         # 修改的目标资源文件
 
configMapGenerator:
- name: example-configmap
  literals:
  - FOO=Bar
EOF

```

所生成的configmap资源文件

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  name: example-configmap-42cfbf598f
```



预览修改之后的yaml文件资源信息，以properties配置文件为例：

```bash
# 预览yaml资源文件命令
# kubectl kustomize ./
apiVersion: v1
data:
  application.properties: |
    FOO=Bar
kind: ConfigMap         		 # 生成一个ConfigMap
metadata:
  name: my-configMap-b992tf4h8f  # ConfigMap资源名称
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: ikubernetes/myapp:v1
        name: myapp
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/conf
          name: my-conf
      volumes:
      - configMap:
          name: my-configMap-b992tf4h8f  # 生成的cm资源
        name: my-conf
```

## secretGenerator

### 基于文件生成Secret

```yaml
# 创建用户密码文件：
cat >my-secret.conf <<EOF
USERNAME=myuser
PASSWORD=mypassword
EOF

# 创建kustomization文件
cat >./kustomization.yaml <<EOF
resources:
  - demo.yaml         	# 修改的目标资源文件

secretGenerator:		# 使用secretGenerator生成配置
- name: example-secret
  files:
  - my-secret.conf
EOF
```

所生生成的secret 资源文件

```yaml
apiVersion: v1
data:
  my-secret.conf: VVNFUk5BTUU9bXl1c2VyClBBU1NXT1JEPW15cGFzc3dvcmQK
kind: Secret
metadata:
  name: example-secret-g2b72m9gk2
type: Opaque
```

### 基于键值对值生成 Secret

```yaml
cat >./kustomization.yaml <<EOF
resources:
  - demo.yaml         # 修改的目标资源文件
  
secretGenerator:
- name: example-secret
  literals:
  - username=admin
  - password=secret
EOF

```

所生成的secret资源文件

```yaml
apiVersion: v1
data:
  password: c2VjcmV0
  username: YWRtaW4=
kind: Secret
metadata:
  name: example-secret-8c5228dkb9
type: Opaque
```



预览修改之后的yaml文件资源信息，以文件为例：

```yaml
# kubectl kustomize ./
apiVersion: v1
data:
  my-secret.conf: VVNFUk5BTUU9bXl1c2VyClBBU1NXT1JEPW15cGFzc3dvcmQK
kind: Secret
metadata:
  name: my-secret-g2b72m9gk2
type: Opaque
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: ikubernetes/myapp:v1
        name: myapp
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/conf/secret
          name: my-secret
      volumes:
      - name: my-secret
        secret:
          secretName: my-secret-g2b72m9gk2
```



## generatorOptions

generatorOptions 所生成的 ConfigMap 和 Secret 都会包含内容哈希值后缀。 这是为了确保内容发生变化时，所生成的是新的 ConfigMap 或 Secret。 要禁止自动添加后缀的行为，用户可以使用 generatorOptions。 除此以外，为生成的 ConfigMap 和 Secret 指定贯穿性选项也是可以的。

```yaml
cat >./kustomization.yaml <<EOF
resources:
  - demo.yaml         # 修改的目标资源文件

configMapGenerator:
- name: example-configmap
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true    # 禁止name自动添加后缀
  labels:
    type: generated
  annotations:
    note: generated
EOF
```

所生成的configmap资源

```yaml
# kubectl kustomize ./ 
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-configmap
```

## 贯穿性字段

在项目中为所有 Kubernetes 对象设置贯穿性字段是一种常见操作。 贯穿性字段的一些使用场景如下：

- 为所有资源设置相同的名字空间
- 为所有对象添加相同的前缀或后缀
- 为对象添加相同的标签集合
- 为对象添加相同的注解集合

kustomization.yaml文件

```yaml
cat >./kustomization.yaml <<EOF
resources:
- demo.yaml

namespace: my-namespace 	# 添加命名空间
namePrefix: dev-         	# name名称添加前缀
nameSuffix: "-001"				# name名称添加后缀
commonLabels:							# 添加标签
  app: myapp
commonAnnotations:				# 添加注解
  oncallPager: 800-555-1212
EOF

```

生成后的Deployment资源文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-myapp-001   				# 添加的前缀后缀
  annotations:
    oncallPager: 800-555-1212   # 注解
  labels:
    app: myapp									# 标签	
  namespace: my-namespace				# 命名空间
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        ports:
        - containerPort: 80
```

## 组织合并

将多个yaml文件进行合并

```yaml
# 创建 kustomization.yaml 来组织以下两个资源
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

## 定制

### patchesStrategicMerge(补丁文件)

可以用来对资源执行不同的定制。 Kustomize 通过 patchesStrategicMerge 和 patchesJson6902 支持不同的打补丁机制。 patchesStrategicMerge 的内容是一个文件路径的列表，其中每个文件都应可解析为 策略性合并补丁（Strategic Merge Patch）。 

注意：**补丁文件中的名称必须与已经加载的资源的名称匹配**。 建议构造规模较小的、仅做一件事情的补丁。 例如，构造一个补丁来增加 Deployment 的副本个数；构造另外一个补丁来设置内存限制。

```yaml
# 创建 deployment.yaml 文件
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

```

```yaml
# 生成一个补丁 increase_replicas.yaml
cat <<EOF > increase_replicas.yaml
apiVersion: apps/v1			
kind: Deployment					# 名称必须与已经加载的资源的名称匹配
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF

```

```yaml
# 生成另一个补丁 set_memory.yaml
cat <<EOF > set_memory.yaml
apiVersion: apps/v1
kind: Deployment					# 名称必须与已经加载的资源的名称匹配
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:				# 添加资源限制
          limits:
            memory: 512Mi
EOF

```

```yaml
# 生成kustomization.yaml文件进行整合
cat >./kustomization.yaml <<EOF
resources:
- deployment.yaml       # 目标文件

patchesStrategicMerge: 		# 加载整合补丁文件
- increase_replicas.yaml
- set_memory.yaml
EOF
```

生成的资源结果

```yaml
# kubectl kustomize ./ 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 512Mi
```

### patchesJson6902 (json补丁)

并非所有资源或者字段都支持策略性合并补丁。为了支持对任何资源的任何字段进行修改， Kustomize 提供通过 patchesJson6902 来应用 JSON 补丁的能力。 

为了给 JSON 补丁找到正确的资源，需要在 kustomization.yaml 文件中指定资源的组（`group`）、 版本（`version`）、类别（`kind`）和名称（`name`）。 例如，为某 Deployment 对象增加副本个数的操作也可以通过` patchesJson6902 `来完成：

```yaml
# 创建一个 deployment.yaml 文件
catF >deployment.yaml <<EO
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v1
        env:
        - name: ENV_VAR_1
          value: value1
```

创建一个path.yaml文件

其中op的操作类型有：

- **add：**添加一个新值。如果指定的路径已存在值，则替换该值；如果路径不存在，则创建路径并设置值。如果要在数组中添加新元素，可以使用 - 作为数组索引，表示将元素添加到数组末尾。
- **remove：**删除指定路径的值。如果路径不存在，该操作将失败。
- **replace：**替换指定路径的值。如果路径不存在，该操作将失败。
- **move：**将一个值从源路径移动到目标路径。源路径和目标路径都必须存在。该操作等效于先执行 remove 操作，然后执行 add 操作。
- **copy：**复制一个值从源路径到目标路径。源路径必须存在，目标路径可以不存在。该操作等效于先获取源路径的值，然后使用 add 操作将其添加到目标路径。
- **test：**测试指定路径的当前值是否与提供的值相等。如果值不相等，该操作将失败。

示例：

```yaml
cat >path.yaml <<EOF
# 增加环境变量
- op: add				# 添加新值
  path: /spec/template/spec/containers/0/env/-      # 对应修改的配置的路径
  value:
    name: ENV_VAR_2
    value: value2

# 替换副本数
- op: replace    # 替换原来值
  path: /spec/replicas
  value: 3
EOF

#image
- op: replace
  path: /spec/template/spec/containers/0/image   # (0 可以解释为代替 - 符号)，表示第一个容器的镜像
  value: myapp:v2
```

kustomization.yaml文件

```yaml
cat >kustomization.yaml <<EOF
resources:
- deployment.yaml       # 目标文件

patchesJson6902:
- target:
    group: apps					# 修改的api资源属于 apps 组
    version: v1
    kind: Deployment
    name: myapp
  path: patch.yaml
EOF

```

所生成的资源

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 3
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2
        env:
        - name: ENV_VAR_1
          value: value1
        - name: ENV_VAR_2     	# 新增一个变量
          value: value2
```

### 容器镜像

kustomization.yaml文件

```yaml
cat > kustomization.yaml <<EOF
resources:
- deployment.yaml       # 目标文件

images:
- name: myapp					# 注意这里对应的是资源文件中的image，而不是name
  newName: myapp1				# 指定新的镜像名称
  newTag: v2					# 新的镜像版本
EOF
```

### vars变量注入

有些时候，Pod 中运行的应用可能需要使用来自其他对象的配置值。例如某 Deploymet 对的 pod 需要从环境变量或命合行参数中读取 Servic的名称。由于在 kustomization.yaml 文件中添加 namePrefix 或 nameSuffix 时 Service 名称可能发生变化，建议不要在命分参数中硬编码Service名称。对于这种使用场景，Kustomize 可以通过 vars 将 Service 名称注入到容器中。

deployment.yaml文件

```yaml
cat >deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      run: my-app
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v1
EOF
```

service.yaml文件

```yaml
cat >service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: ClusterIP
  ports:
  - name: http-my-app
    port: 80
    protocol: TCP
    targetPort: 80
EOF
```

kustomization.yaml文件

```yaml
cat >kustomization.yaml <<EOF
resources:
- deployment.yaml       # 目标文件
- service.yaml

namePrefix: dev-         	# name名称添加前缀
nameSuffix: "-001"				# name名称添加后缀

vars: 
- name: MY_SERVICE_NAME
  objref:									# 变量指向哪个地方
    kind: Service
    name: my-app
    apiVerion: v1
EOF
```

## 基准(bases)与覆盖(overlays)

Kustomize中有 基准(bases)和覆盖(overlays) 的概念区分。

基准是包含 `kustomization.yam` 文件的一个目录，其中包含一组资源及其相关的定制。基准可以是本地目录或者来自远程仓库的目录，只要其中存在 `kustomaton.yaml` 文件即可。

覆盖也是一个目录，其中包含将其他`kustomization` 目录当做bases 来引用的 `kustomization.yam` 文件。基准不了解覆盖的存在，且可被多个覆盖所使用。覆盖则可以有多个基准，且可针对所有基准中的资源执行组织操作，还可以在其上执行定制。

整体的目录结构大概如下：

```bash
.
└── base
    ├── deployment.yaml
    ├── service.yaml
    ├── kustomization.yaml
├── overlays
│   ├── sit
│   │   ├── path1.yaml
│   │   ├── path2.yaml
│   │   └── application.properties
│   │   └── kustormization.yaml
│   ├── prod
│   │   ├── path1.yaml
│   │   ├── path2.yaml
│   │   └── application.properties
│   │   └── kustormization.yaml
│   └── dev
│   │   ├── path1.yaml
│   │   ├── path2.yaml
│   │   └── application.properties
│   │   └── kustormization.yaml
```



创建一个包含基准的目录

```bash
mkdir base
```

并才创建基础资源文件

deployment.yaml文件

```yaml
cat >deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      run: my-app
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v1
				command: ["start","--host","${MY_SERVICE_NAME}"]
EOF

```

service.yaml

```yaml
cat >service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: ClusterIP
  ports:
  - name: http-my-app
    port: 80
    protocol: TCP
    targetPort: 80
EOF

```

基准kustomization.yaml文件

```yaml
cat >kustomization.yaml <<EOF
resources:
- deployment.yaml       # 目标文件
- service.yaml
EOF

```

此基准可在多个覆盖中使用。可以在不同的覆盖中添加不同的 namePrefix或 其他贯穿性字段。下面是两个使用同一基准的覆盖:

创建一个overlays目标，并且在下面创建各个环境的配置的目录，起到区分环境的作用，各个环境互不影响

```bash
mkdir overlays/{dev,prod} -p
```

- overlays/dev目录创建dev环境的kustomization.yml资源文件

```yaml
cat >overlays/dev/kustomization.yml <<EOF
bases:							# 引用bases
- ../../base				# 指定base的路径

namePrefix: dev-    # 资源名称添加dev环境前缀
# 添加其他逻辑，直接往下写即可
EOF

```

- overlays/prod目录下创建prod环境的kustomization.yml资源文件

```yaml
cat >overlays/prod/kustomization.yml <<EOF
bases:
- ../../base				# 指定base的路径

namePrefix: prod-    # 资源名称添加dev环境前缀
# 添加其他逻辑，直接往下写即可
EOF

```

