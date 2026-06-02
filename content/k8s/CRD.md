+++
title = "CRD"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 3
+++

官方文档：[https://kubernetes.io/zh-cn/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/](https://kubernetes.io/zh-cn/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

# 介绍

CRD（CustomResourceDefinition，自定义资源定义）是 Kubernetes 最主流的 API 扩展机制。通过它，用户可以在集群中注册新的资源类型，使 Kubernetes 原生支持这些对象的 CRUD 操作。它让开发者无需修改核心代码，即可声明新资源类型，结合控制器实现领域自动化。

​  

Kubernetes 的核心思想是“声明式 API”。当你创建一个 CRD 时，本质上是告诉 API Server：“为我注册一个新的资源类型”

API Server 会自动生成该类型的 REST 接口，并提供以下支持：

-   CRUD（创建、查询、更新、删除）接口
-   OpenAPI Schema 校验
-   RBAC 权限管理
-   kubectl 命令支持

# 资源结构

## 创建资源

官方的示例：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # API组名称，用于 REST API：/apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是首字母大写，单数形式的驼峰命名（CamelCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
    - cront
```

创建：

```yaml
kubectl apply -f resourcedefinition.yaml

#查看
kubectl get crd|grep cron
crontabs.stable.example.com                    2026-05-22T06:24:20Z
```

## 创建自定义资源对象

```yaml
# 创建资源的时候的定义的group信息
apiVersion: "stable.example.com/v1"  
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  # schema中定义的字段
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

创建

```yaml
kubectl apply -f my-crontab.yaml
```

## 版本管理与转换（Versioning & Conversion）

官文：[https://kubernetes.io/zh-cn/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/](https://kubernetes.io/zh-cn/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)

支持多版本声明：

```yaml
versions:
  - name: v1alpha1
    served: true
    storage: false
  - name: v1
    served: true
    storage: true
```

在创建之后，API 服务器开始在 HTTP REST 端点上为每个已启用的版本提供服务。 在上面的示例中，API 版本可以在 `/apis/example.com/v1beta1` 和 `/apis/example.com/v1` 处获得

## Schema 与验证

CRD 使用 OpenAPI v3 Schema 来约束资源字段，保证集群中自定义资源的结构化一致性。

```yaml
schema:
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        required: ["engine", "version"]
        properties:
          engine:
            type: string
            enum: ["mysql", "postgres"]
          version:
            type: string
```

常用功能包括：

-   `enum`：限定值范围
-   `required`：指定必填字段
-   `default`：设定默认值
-   `nullable`：允许空值

## 子资源（Subresources）

定制资源支持子资源：

-   `/status`
-   `/scale`

通过在 CustomResourceDefinition 中定义 `status` 和 `scale`， 可以有选择地启用这些子资源。

```yaml
subresources:
  status: {}
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.replicas
```

可以通过标准方式更新状态或伸缩资源：

```yaml
kubectl scale database xxx --replicas=3
```
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
            status:
              type: object
              properties:
                replicas:
                  type: integer
                labelSelector:
                  type: string
      # subresources 描述定制资源的子资源
      subresources:
        # status 启用 status 子资源
        status: {}
        # scale 启用 scale 子资源
        scale:
          # specReplicasPath 定义定制资源中对应 scale.spec.replicas 的 JSON 路径
          specReplicasPath: .spec.replicas
          # statusReplicasPath 定义定制资源中对应 scale.status.replicas 的 JSON 路径 
          statusReplicasPath: .status.replicas
          # labelSelectorPath  定义定制资源中对应 scale.status.selector 的 JSON 路径 
          labelSelectorPath: .status.labelSelector
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

# 适用场景：

典型场景：

-   应用控制器（如 MySQL Operator、Kafka Operator）
-   平台级自动化资源（如 JobTemplate、Pipeline）
-   AI 任务编排（如 InferenceJob、ModelDeployment）
-   观测与策略扩展（如 LogPolicy、AlertRule）
