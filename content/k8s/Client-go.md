+++
title = "Client-go"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 2
+++

# 简介

Client-Go 是负责与 Kubernetes APIServer 服务进行交互的客户端库，利用 Client-Go 与 KubernetesAPIServer 进行的交互访问，以此来对 Kubernetes 中的各类资源对象进行管理操作，包括内置的资源对象及CRD。

Client-Go 不仅被 Kubernetes 项目本身使用，其它围绕着 Kubernetes 的生态，也被大量的使用，例如: kubectl、ETCD-operator等等；

**​**  

**Kubernetes 的 API接口规则：**

在围绕Kubernetes 的功能开发中，经常会调用到 Kubernetes 提供的API。Kubernetes API 是通过 HTTP 协议以 RESTful的形式提供的，同时支持JSON和 Protobuf 的数据格式，Protobuf 是为方便集群内部调用而支持的。我们自己平时调用 Kubernetes 接口，一般都是使用JSON数据格式的。

在KubernetesAPI中，我们一般使用GVR 或GVK 来区分特定的资源即根据不同的分组、版本及资源，进行 URL的定义，有了分组和多版本的支持，即便是后续的版本中，需要去掉资源对象的某些字段或者重构API 资源，也可以保证版本之间的兼容性。

同时，因为历史原因，因此 Kubernetes 的 API 分组，是分为 `无组名资源组` 和`有组名资源组`，无组名资源组 也被称之为核心资源组，`Core Group`。

![image.png](assets/Client-go-1.png)

![image.png](assets/Client-go-2.png)

**GVR/GVK 含义介绍**

-   G(Group) : 资源组，包含一组资源操作的集合。
-   V (Version): 资源版本，用于区分不同 API的稳定程度及兼容性
-   R(Resource) : 资源信息，用于区分不同的资源 API。
-   K(Kind): 资源对象的类型，每个资源对象都需要 Kind 来区分它自身代表的资源类型

一般在接口调用的时候，我们只需要知道 GVR 即可。通过 GVR 操作对应的资源对象。

通过GVR 组合成 RESTful APl 请求路径例如，针对apps/1下面 Deployment 的 RESTful API 请求路径如下所示:

```bash
GET /apis/apps/v1/namespaces/{namespace}/deployments/{name}
```

GVK，相反，通过GVK信息则可以获取要读取的资源对象的GVR，进而构建RESTful API请求获取对应的资源。

这种 GVK与GVR 的映射叫做 RESTMapper。RESTMapper其主要作用是在 Lister-Watcher 时，根据 Schema 定义的类型GVK 解析出GVR，向APIServer 发起HTTP 请求获取资源，然后Watch。

# 客户端对象

Client-Go 共提供了 4种与 Kubernetes APIServer 交互的客户端对象。分别是 RESTClient、DiscoveryClient、ClientSet、DynamicClient。

-   RESTClient：最基础的客户端，主要是对 HTTP 请求进行了封装，支持Json 和 Protobuf格式的数据。
-   DiscoveryClient：发现客户端，负责发现APIServer 支持的资源组、资源版本和资源信息的。
-   ClientSet：负责操作 Kubernetes 内置的资源对象，例如: Pod、Service等。
-   DynamicClient：动态客户端，可以对任意的 Kubernetes 资源对象进行通用操作，包括 CRD

![image.png](assets/Client-go-3.png)

  

因为client-go是第三方依赖包，并不是go的默认依赖包，所以需要进行安装

安装k8s客户端依赖包

```go
// 一般装这两个就够，其他也都自动装完
go get -u "k8s.io/client-go/kubernetes"
go get -u "k8s.io/api/apps/v1"
```

## RESTClient

RESTClient 是所有客户端的父类，这也是为啥前面说，它是最基础的客户端的原因。

它提供了 RESTful 对应的方法的封装，如：Get()、Put()、Post()、Delete() 等。通过这些封装发方法与 Kubernetes APIServer RESTful API 进行交互。

因为它是所有客户端的父类，所以它可以操作 Kubernetes 内置的所有资源对象以及 CRD。

示例：

```go
package main

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// 加载配置文件，生成 config 对象
	config, err := clientcmd.BuildConfigFromFlags("", "kube/config")
	if err != nil {
		panic(err.Error())
	}

	// 配置 API 路径，根路径之后的地址
	config.APIPath = "api" //pod：api/v1/pods
	// config.APIPath = "apis"  //deployment: apis/apps/v1/namespaces/{namespace}/deployment/{deployment}

	// 配置分组版本(核心资源组里面的资源)
	config.GroupVersion = &corev1.SchemeGroupVersion // 无名资源组，group: "", version: "v1"

	// 配置数据的编解码器
	config.NegotiatedSerializer = scheme.Codecs

	// 实例化 RESTClient
	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err.Error())
	}

	// 定义返回接收值
	pods := &corev1.PodList{}
	svcs := &corev1.ServiceList{}

	err = restClient.Get().
		Namespace("kube-system").          // 查询的 Namespace
		Resource("pods").                 // 查询的核心资源组资源类型：pods、services、nodes、namespaces、secrets、configmaps、endpoints等
		VersionedParams(&metav1.ListOptions{Limit: 100}, scheme.ParameterCodec). // 参数及序列化工具
		Do(context.TODO()).                                                      // 发送请求
		Into(pods)                                                               // 写入返回值
	if err != nil {
		panic(err.Error())
	}

	err = restClient.Get().
		Namespace("kube-system").                                                // 查询的 Namespace
		Resource("services").                                                    // 查询的核心资源组资源类型
		VersionedParams(&metav1.ListOptions{Limit: 100}, scheme.ParameterCodec). // 参数及序列化工具
		Do(context.TODO()).                                                      // 发送请求
		Into(svcs)                                                               // 写入返回值
	if err != nil {
		panic(err.Error())
	}

	// 输出返回结果
	fmt.Println("pods:")
	for _, p := range pods.Items {
		fmt.Printf("namespace: %v, name: %v, status: %v\n", p.Namespace, p.Name, p.Status.Phase)
	}
	fmt.Println("svcs:")
	for _, s := range svcs.Items {
		fmt.Printf("namespace: %v, name: %v, status: %v\n", s.Namespace, s.Name, s.Spec.ClusterIP)
	}
}
```

输出：

```go
pods:
namespace: kube-system, name: calico-kube-controllers-595b58b579-5xmkw, status: Running
namespace: kube-system, name: calico-node-72hxj, status: Running
namespace: kube-system, name: calico-node-txq8j, status: Running
namespace: kube-system, name: coredns-6d8c4cb4d-bvlkq, status: Running
namespace: kube-system, name: coredns-6d8c4cb4d-hg7pm, status: Running
namespace: kube-system, name: etcd-k8s-master1, status: Running
namespace: kube-system, name: kube-apiserver-k8s-master1, status: Running
namespace: kube-system, name: kube-controller-manager-k8s-master1, status: Running
namespace: kube-system, name: kube-proxy-hfznn, status: Running
namespace: kube-system, name: kube-proxy-pjz5j, status: Running
namespace: kube-system, name: kube-scheduler-k8s-master1, status: Running
svcs:
namespace: kube-system, name: kube-dns, status: 10.1.0.10
```

## ClientSet

ESTClient，它虽然可以操作 Kubernetes 的所有资源对象，但是使用起来确实比较复杂，需要配置的参数过于繁琐，因此，为了更优雅的更方便的与 Kubernetes APIServer 进行交互，则需要进一步的封装。

ClientSet 是基于 RESTClient 的封装，同时 ClientSet 是使用预生成的 API 对象与 APIServer 进行交互的，这样做更方便进行二次开发。

ClientSet 是一组资源对象客户端的集合，例如负责操作 Pods、Services 等资源的 CoreV1Client，负责操作 Deployments、DaemonSets 等资源的 AppsV1Client 等。通过这些资源对象客户端提供的操作方法，即可对 Kubernetes 内置的资源对象进行 Create、Update、Get、List、Delete 等操作。

```go
package main

import (
    "context"
    "fmt"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 加载配置文件，生成 config 对象
    config, err := clientcmd.BuildConfigFromFlags("", "../../kubeconfig")
    if err != nil {
        panic(err.Error())
    }

    // 实例化 ClientSet
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err.Error())
    }

    // 查询 default 下的 pods 部门资源信息
    pods, err := clientset.
        CoreV1().                                  // 实例化资源客户端，这里标识实例化 CoreV1Client
        Pods("default").                           // 选择 namespace，为空则表示所有 Namespace
        List(context.TODO(), metav1.ListOptions{}) // 查询 pods 列表
    if err != nil {
        panic(err.Error())
    }

    // 输出 Pods 资源信息
    for _, item := range pods.Items {
        fmt.Printf("namespace: %v, name: %v\n", item.Namespace, item.Name)
    }
}
```

## DynamicClient

DynamicClient 是一种动态客户端，通过动态指定资源组、资源版本和资源等信息，来操作任意的 Kubernetes 资源对象的一种客户端。即不仅仅是操作 Kubernetes 内置的资源对象，还可以操作 CRD。这也是与 ClientSet 最明显的一个区别。

使用 ClientSet 的时候，程序会将所用的版本与类型紧密耦合。而 DynamicClient 使用嵌套的 map[string]interface{} 结构存储 Kubernetes APIServer 的返回值，使用反射机制，在运行的时候，进行数据绑定，这种方式更加灵活，但是却无法获取强数据类型的检查和验证。

此外，在介绍 DynamicClient 之前，还需要了解另外两个重要的知识点，Object.runtime 接口和 Unstructured 结构体。

-   **Object.runtime：**Kubernetes 中的所有资源对象，都实现了这个接口，其中包含 DeepCopyObject 和 GetObjectKind 的方法，分别用于对象深拷贝和获取对象的具体资源类型。
-   **Unstructured**：包含 map[string]interface{} 类型字段，在处理无法预知结构的数据时，将数据值存入 interface{} 中，待运行时利用反射判断。该结构体提供了大量的工具方法，便于处理非结构化的数据。

示例：

```go
package main

import (
    "context"
    "fmt"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/client-go/dynamic"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 加载配置文件，生成 config 对象
    config, err := clientcmd.BuildConfigFromFlags("", "../../kubeconfig")
    if err != nil {
        panic(err.Error())
    }

    // 实例化 DynamicClient
    dynamicClient, err := dynamic.NewForConfig(config)
    if err != nil {
        panic(err.Error())
    }

    // 设置要请求的 GVR
    gvr := schema.GroupVersionResource{
        Group:    "",
        Version:  "v1",
        Resource: "pods",
    }

    // 发送请求，并得到返回结果
    /*
        请求过程
        Resource：基于gvr生成一个针对于资源的客户端，也可以称之为动态客户端，dynamicResourceClient
        Namespace：指定一个可操作的命名空间，同时它是dynamicResourceClient的方法
        List：首先是通过RESTClient调用APIServer的接口，返回pod的数据，返回的数据格式是二进制的json格式，然后通过一些列的解析转换成unstructtrued.unstructtruedlist类型数据
    */
    unStructData, err := dynamicClient.Resource(gvr).Namespace("default").List(context.TODO(), metav1.ListOptions{})
    if err != nil {
        panic(err.Error())
    }

    // 使用反射将 unStructData 的数据转成对应的结构体类型，例如这是是转成 v1.PodList 类型
    podList := &corev1.PodList{}
    err = runtime.DefaultUnstructuredConverter.FromUnstructured(
        unStructData.UnstructuredContent(),
        podList,
    )
    if err != nil {
        panic(err.Error())
    }

    // 输出 Pods 资源信息
    for _, item := range podList.Items {
        fmt.Printf("namespace: %v, name: %v\n", item.Namespace, item.Name)
    }
}
```

## DiscoveryClient

前面介绍的 3 种客户端，都是针对与资源对象管理的。而 DiscoveryClient 则是针对于资源的。用于查看当前 Kubernetes 集群支持那些资源组、资源版本、资源信息。

示例：

```go
package main

import (
    "fmt"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/client-go/discovery"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 加载配置文件，生成 config 对象
    config, err := clientcmd.BuildConfigFromFlags("", "kube/kubeconfig")
    if err != nil {
        panic(err.Error())
    }

    // 实例化 DiscoveryClient
    discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
    if err != nil {
        panic(err.Error())
    }

    _, apiResources, err := discoveryClient.ServerGroupsAndResources()
    if err != nil {
        panic(err.Error())
    }

    for _, list := range apiResources {
        gv, err := schema.ParseGroupVersion(list.GroupVersion)
        if err != nil {
            panic(err.Error())
        }
        for _, resource := range list.APIResources {
            fmt.Printf("name: %v, group: %v, version: %v\n", resource.Name, gv.Group, gv.Version)
        }
    }
}
```

输出：

```go
name: bindings, group: , version: v1
name: componentstatuses, group: , version: v1
name: configmaps, group: , version: v1
name: endpoints, group: , version: v1
name: events, group: , version: v1
name: limitranges, group: , version: v1
name: namespaces, group: , version: v1
name: namespaces/finalize, group: , version: v1
name: namespaces/status, group: , version: v1
... ...
```

# ClientSet实战

## 查资源

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	appsv1 "k8s.io/api/apps/v1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/retry"
    "k8s.io/client-go/util/homedir"
	"os"
	"path/filepath"
)

func main() {
	var kubeconfig *string
	// client-go包提供的方法，获取家目录
	if home := homedir.HomeDir(); home != "" {
        // 拼接取config文件：~/.kube/config
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file)")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "a string")
	}

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

    // 获取命名空间
	namespaceList, err := clientset.CoreV1().Namespaces().List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	for _, n := range namespaceList.Items {
		fmt.Printf("* %s\n", n.Name)
	}

    // 获取namespace下的pod
	podlist, err := clientset.CoreV1().Pods("test").List(context.Background(),metav1.ListOptions{})
	if err !=nil {
		panic(err)
	}
	for _, p := range podlist.Items {
		fmt.Printf("* %s\n", p.Name)
	}

    // 获取namespace下的svc
    svcList, err := clientset.CoreV1().Services("test").List(context.TODO(), metav1.ListOptions{})
    	for _, p := range svcList.Items {
		fmt.Printf("* %s\n", p.Name)
	}
}
```

## 管理命名空间

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	appsv1 "k8s.io/api/apps/v1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/retry"
	"os"
	"path/filepath"
)

func main() {
	var kubeconfig *string
	// client-go包提供的方法，获取家目录
	if home := homedir.HomeDir(); home != "" {
        // 拼接取config文件：~/.kube/config
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file)")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "a string")
	}
	flag.Parse()
    
    // 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
    // 创建命名空间
	ns := &apiv1.Namespace{
		ObjectMeta: metav1.ObjectMeta{
			Name: "anyTest",
		},
	}
	res, err := clientset.CoreV1().Namespaces().Create(context.TODO(), ns, metav1.CreateOptions{})
	if err != nil {
		panic(err)
	}
	fmt.Printf("Created namespace %s.\n", res.GetObjectMeta().GetName())

    // 更新命名空间
    namespace := &corev1.Namespace{
        // 根据需要的信息进行修改
		ObjectMeta: metav1.ObjectMeta{
			Name:                       "zhangpeng",
			GenerateName:               "",
			Namespace:                  "",
			SelfLink:                   "",
			UID:                        "",
			ResourceVersion:            "",
			Generation:                 0,
			CreationTimestamp:          metav1.Time{},
			DeletionTimestamp:          nil,
			DeletionGracePeriodSeconds: nil,
			Labels: map[string]string{
				"dev": "test",
			},
			Annotations:     nil,
			OwnerReferences: nil,
			Finalizers:      nil,
			ClusterName:     "",
			ManagedFields:   nil,
		},
	}
	result, _ := clientset.CoreV1().Namespaces().Update(context.TODO(), namespace, metav1.UpdateOptions{})
	fmt.Println(result)
    
    // 删除命名空间
    NamespaceName := "test"
	if err = clientset.CoreV1().Namespaces().Delete(context.TODO(), NamespaceName, metav1.DeleteOptions{}); err != nil {
		fmt.Println(err.Error())
		return
	} else {
		fmt.Printf("Deleted Namespaces %s\n", NamespaceName)
	}
}
```

## 管理deployment

### 代码中直接定义创建

使用client客户端创建一个deployment资源示例:

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	appsv1 "k8s.io/api/apps/v1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"k8s.io/client-go/util/retry"
	"os"
	"path/filepath"
)

func main() {
	var kubeconfig, namespace, name *string
	// 使用客户端util工具包中的homedir方法获取kubeconfig文件路径
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file)")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "a string")
	}
	// 获取namespace和deployment的名称参数
	namespace = flag.String("namespace", "default", "(optional) namespace to use")
	name = flag.String("name", "demo-deployment", "(optional) name of the deployment")
	flag.Parse()

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
	// 创建deployment的客户端对象
	deploymentClient := clientset.AppsV1().Deployments(*namespace)

	//定义deployment的配置信息
	deployment := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name: *name,
		},
		// 定义deployment的配置信息
		Spec: appsv1.DeploymentSpec{
			Replicas: int32Ptr(2),
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": "demo",
				},
			},
			// 定义pod的配置信息
			Template: apiv1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						// 定义pod的标签,与deployment的标签保持一致
						"app": "demo",
					},
				},
				// 定义容器相关的配置信息
				Spec: apiv1.PodSpec{
					// 容器列表
					Containers: []apiv1.Container{
						{
							Name:  "nginx", // 容器名称
							Image: "nginx", // 容器使用的镜像
							// 定义容器需要暴露的端口信息
							Ports: []apiv1.ContainerPort{
								{
									Name:          "http",
									Protocol:      apiv1.Protocol("TCP"),
									ContainerPort: 80,
								},
							},
						},
					},
				},
			},
		},
	}
	// 创建deployment
	fmt.Println("creating deployment...")
	result, err := deploymentClient.Create(context.TODO(), deployment, metav1.CreateOptions{})
	if err != nil {
		panic(err)
	}
	// 打印创建的deployment信息
	fmt.Printf("Created deployment %s.\n", result.GetObjectMeta().GetName())
	// 监听输入，进行阻塞
	prompt()
	// 更新deployment
	fmt.Println("Updating deployment...")
	// 使用客户端工具提供的方法，处理资源冲突问题，遇到自动捕获这些冲突并重试更新操作，直到成功或达到最大重试次数（当多个客户端同时尝试更新同一个资源时，可能会发生冲突）
	retryErr := retry.RetryOnConflict(retry.DefaultRetry, func() error {
		result, err := deploymentClient.Get(context.TODO(), *name, metav1.GetOptions{})
		if err != nil {
			panic(fmt.Errorf("Failed to get lastest version of deployment: %v", err))
		}
		result.Spec.Replicas = int32Ptr(1)                             // 更改副本数
		result.Spec.Template.Spec.Containers[0].Image = "nginx:1.19.2" //更改对应容器镜像版本
		// 更新deployment
		_, updateErr := deploymentClient.Update(context.TODO(), result, metav1.UpdateOptions{})
		return updateErr
	})
	if retryErr != nil {
		panic(fmt.Errorf("Update failed: %v", retryErr))
	}
	fmt.Println("Updated deployment...")

	// 监听输入，进行阻塞
	prompt()
	fmt.Printf("Listing deployments in namespace: %s\n", *namespace)
	// 列出当前namespace下的所有deployment资源
	list, err := deploymentClient.List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	for _, d := range list.Items {
		fmt.Printf("* %s (%d replicas)\n", d.Name, *d.Spec.Replicas)
	}

	prompt()
	// 删除deployment
	fmt.Println("Deleting deployment...")
	// 创建删除对象
	deletePolicy := metav1.DeletePropagationForeground
	// 进行deployment资源删除操作
	if err := deploymentClient.Delete(context.TODO(), *name, metav1.DeleteOptions{
		PropagationPolicy: &deletePolicy,
	}); err != nil {
		panic(err)
	}
	fmt.Println("Deleted deployment.")

}

// int32Ptr 返回一个指向 int32 值的指针
func int32Ptr(i int32) *int32 {
	return &i
}

// 停顿输入函数
func prompt() {
	fmt.Printf("-> Press Return key to continue.")
	// 创建一个输入对象
	scanner := bufio.NewScanner(os.Stdin)
	// 读取用户输入
	for scanner.Scan() {
		break
	}
	if err := scanner.Err(); err != nil {
		panic(err)
	}
	fmt.Println()

}
```

### 读取yaml文件创建

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
    "encoding/json"
	appsv1 "k8s.io/api/apps/v1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/retry"
	"os"
	"path/filepath"
)

func main() {
	var kubeconfig *string
	// client-go包提供的方法，获取家目录
	if home := homedir.HomeDir(); home != "" {
        // 拼接取config文件：~/.kube/config
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file)")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "a string")
	}
	flag.Parse()
    
    // 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
    // 读取yaml文件
    n, err := ioutil.ReadFile("tmp/nginx.yaml")
	if err != nil {
		panic(err)
	}
    // 创建Deployment配置空对象
	nginxDep := &appv1.Deployment{}
    // 使用yaml方法载入文件信息
	nginxYaml, _ := yaml.ToJSON(n)
	if err = json.Unmarshal(nginxYaml, nginxDep); err != nil {
		return
	}
    // 创建
    if result, err = clientset.AppsV1().Deployments("test").Create(context.Background(), nginxDep, metav1.CreateOptions{}); err != nil {
        fmt.Println(err)
        return
	}
    // 打印创建的deployment信息
	fmt.Printf("Created deployment %s.\n", result.GetObjectMeta().GetName())
}
```

## 管理demoset

创建并管理demoset资源

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	appsv1 "k8s.io/api/apps/v1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"k8s.io/client-go/util/retry"
	"os"
	"path/filepath"
)

func main() {
	var kubeconfig, namespace, name *string
	// 使用客户端util工具包中的homedir方法获取kubeconfig文件路径
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file)")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "a string")
	}
	// 获取namespace和deployment的名称参数
	namespace = flag.String("namespace", "default", "(optional) namespace to use")
	name = flag.String("name", "demo-deployment", "(optional) name of the deployment")
	flag.Parse()

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
	// 创建deployment的客户端对象
	demosetClient := clientset.AppsV1().DaemonSets(*namespace)

	//定义deployment的配置信息
	deployment := &appsv1.DaemonSet{
		ObjectMeta: metav1.ObjectMeta{
			Name: *name,
		},
		// 定义deployment的配置信息
		Spec: appsv1.DaemonSetSpec{
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": "demo",
				},
			},
			// 定义pod的配置信息
			Template: apiv1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						// 定义pod的标签,与deployment的标签保持一致
						"app": "demo",
					},
				},
				// 定义容器相关的配置信息
				Spec: apiv1.PodSpec{
					// 容器列表
					Containers: []apiv1.Container{
						{
							Name:  "nginx", // 容器名称
							Image: "nginx", // 容器使用的镜像
							// 定义容器需要暴露的端口信息
							Ports: []apiv1.ContainerPort{
								{
									Name:          "http",
									Protocol:      apiv1.Protocol("TCP"),
									ContainerPort: 80,
								},
							},
						},
					},
				},
			},
		},
	}
	// 创建deployment
	fmt.Println("creating deployment...")
	result, err := demosetClient.Create(context.TODO(), deployment, metav1.CreateOptions{})
	if err != nil {
		panic(err)
	}
	// 打印创建的deployment信息
	fmt.Printf("Created deployment %s.\n", result.GetObjectMeta().GetName())
	// 监听输入，进行阻塞
	prompt()
	// 更新deployment
	fmt.Println("Updating deployment...")
	// 使用客户端工具提供的方法，处理资源冲突问题，遇到自动捕获这些冲突并重试更新操作，直到成功或达到最大重试次数（当多个客户端同时尝试更新同一个资源时，可能会发生冲突）
	retryErr := retry.RetryOnConflict(retry.DefaultRetry, func() error {
		result, err := demosetClient.Get(context.TODO(), *name, metav1.GetOptions{})
		if err != nil {
			panic(fmt.Errorf("Failed to get lastest version of deployment: %v", err))
		}
		result.Spec.Template.Spec.Containers[0].Image = "nginx:1.19.2" //更改对应容器镜像版本
		// 更新deployment
		_, updateErr := demosetClient.Update(context.TODO(), result, metav1.UpdateOptions{})
		return updateErr
	})
	if retryErr != nil {
		panic(fmt.Errorf("Update failed: %v", retryErr))
	}
	fmt.Println("Updated deployment...")

	// 监听输入，进行阻塞
	prompt()
	fmt.Printf("Listing deployments in namespace: %s\n", *namespace)
	// 列出当前namespace下的所有deployment资源
	list, err := demosetClient.List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	for _, d := range list.Items {
		fmt.Printf("* %s\n", d.Name)
	}

	prompt()
	// 删除deployment
	fmt.Println("Deleting deployment...")
	// 创建删除对象
	deletePolicy := metav1.DeletePropagationForeground
	// 进行deployment资源删除操作
	if err := demosetClient.Delete(context.TODO(), *name, metav1.DeleteOptions{
		PropagationPolicy: &deletePolicy,
	}); err != nil {
		panic(err)
	}
	fmt.Println("Deleted deployment.")
}

// int32Ptr 返回一个指向 int32 值的指针
func int32Ptr(i int32) *int32 {
	return &i
}

// 停顿输入函数
func prompt() {
	fmt.Printf("-> Press Return key to continue.")
	// 创建一个输入对象
	scanner := bufio.NewScanner(os.Stdin)
	// 读取用户输入
	for scanner.Scan() {
		break
	}
	if err := scanner.Err(); err != nil {
		panic(err)
	}
	fmt.Println()

}
```

# 简单结合Gin

## 准备工作

gin安装

```go
go get github.com/gin-gonic/gin
```

**将客户端单独封装成一个方法文件：**

**1、只封装ClientSet**

创建目录及文件：`src/lib/K8sClient.go`

```go
package lib

import (
	"flag"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"path/filepath"
)

var Config *rest.Config

var K8sClient *kubernetes.Clientset

func init() {
	var (
		kubeconfig *string
		err        error
	)
	// 获取kubeconfig文件路径
	//if home := homedir.HomeDir(); home != "" {
	//	kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")
	//} else {
	//	kubeconfig = flag.String("kubeconfig", "", "a string")
	//}
	//flag.Parse()
	kubeconfig = flag.String("kubeconfig", filepath.Join("kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")

	// 构建配置对象
	Config, err = clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(Config)
	if err != nil {
		panic(err)
	}
	K8sClient = clientset
}
```

在实际工程文件直接导入，就可以进行使用：

```go
import (
	. "k8s-demo1/src/lib"   // 使用.，引用时可以直接引用方法：K8sClient
    // "k8s-demo1/src/lib"  // 这种导入的话，引用：lib.K8sClient
)
```

**2、封装多个客户端**

**创建目录及文件：**`**src/lib/client.go**`

```go
package lib

import (
	"flag"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"path/filepath"
)

// 定义个结构体存放客户端对象
type Clients struct {
	clientSet       kubernetes.Interface
	discoveryClient discovery.DiscoveryInterface
	dynamicClient   dynamic.Interface
}

func NewClients() (clients Clients) {
	var kubeconfig *string
	// 获取kubeconfig文件路径
	//if home := homedir.HomeDir(); home != "" {
	//	kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")
	//} else {
	//	kubeconfig = flag.String("kubeconfig", "", "a string")
	//}
	//flag.Parse()
	kubeconfig = flag.String("kubeconfig", filepath.Join("kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 实例化各种客户端对象
	// 1、实例化客户端对象clientset
	clients.clientSet, err = kubernetes.NewForConfig(config)
	if err != nil {
		return
	}

	// 2、实例化discovery客户端对象
	clients.discoveryClient, err = discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		return
	}

	// 3、实例化 DynamicClient
	clients.dynamicClient, err = dynamic.NewForConfig(config)
	if err != nil {
		return
	}
	return
}

// 因为在Clients中定义的是私有的，所以需要暴露客户端对象
func (c *Clients) ClientSet() kubernetes.Interface {
	return c.clientSet
}
func (c *Clients) DiscoveryClient() discovery.DiscoveryInterface {
	return c.discoveryClient
}
func (c *Clients) DynamicClient() dynamic.Interface {
	return c.dynamicClient
}
```

调用示例:

```go
import (
	// . "k8s-demo1/src/lib"   // 使用.，引用时可以直接引用方法
    "k8s-demo1/src/lib"  // 这种导入的话，引用：lib.client
)

func main() {
    var (clients lib.Clients)

    // 加载客户端
    clients = lib.NewClients()
    clintSet := clients.ClientSet()
    // discoveryClient := clients.DiscoveryClient()
    // dynamicClient := clients.DynamicClient()
}
```

## 获取资源信息

### 简单list

1.  在src目录下创建service目录，当做方法目录，并在下面创建：

1.  **namespace**

src/service/Namespace.go

```go
package service

import (
	"context"
	"github.com/gin-gonic/gin"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	. "k8s/src/lib"
)

func ListNamespace(g *gin.Context) {
	ns, err := K8sClient.CoreV1().Namespaces().List(context.Background(), metav1.ListOptions{})
	if err != nil {
		g.Error(err)
		return
	}
	g.JSON(200, ns)      // 可直接返回，会返回namespace的所有信息，内容数据较多
}
```

2.  **pod**

src/service/Pod.go

```go
package service

import (
	"context"
	"github.com/gin-gonic/gin"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	. "k8s/src/lib"
)

func ListPod(g *gin.Context) {
    // 获取namespace请求参数
	ns := g.Query("namespace")
	podlist, err := K8sClient.CoreV1().Pods(ns).List(context.Background(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	g.JSON(200, podlist)
}
```

3.  deployment

同理

4.  svc

同理

2.  **创建main.go文件**

**项目主入口，设置路由**

```go
package main

import (
	"github.com/gin-gonic/gin"
	"k8s/src/service"
)

func main() {
	g := gin.Default()
	g.GET("/ns", service.ListNamespace)
    g.GET("/dep", service.ListDeployment)
	g.GET("/pod", service.ListPod)
	g.GET("/svc", service.ListSvc)
	g.Run(":8080")
}
```

整体文件目录：

![image.png](assets/Client-go-4.png)

可以命令行控制台go run main.go 也可以goland直接run main.go

3.  **网页请求：**

[http://127.0.0.1:8080/ns](http://127.0.0.1:8080/pod?namespace=kube-system)

![image.png](assets/Client-go-5.png)

返回namespace的所有元数据

### 深入List

前面在获取数据的时候是返回的所有的相关元数据，当我们需要只获取部分数据的时候，就需要对数据进行一定的处理，例如`kubectl get ns`返回的就只有这些数据：

![image.png](assets/Client-go-6.png)

1.  namespace

src/service/Namespace.go

```go
package service

import (
	"context"
	"github.com/gin-gonic/gin"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	. "k8s/src/lib"
	"time"
)

type Time struct {
	// 嵌入了标准库中的 time.Time 类型，并添加了一个标签 protobuf:"-"。这个标签的作用是告诉 Protocol Buffers 编译器忽略这个字段，不将其序列化为 Protocol Buffers 消息的一部分
	time.Time `protobuf:"-"`
}

// 定义一个namespace 结构体
type Namespace struct {
	Name       string `json:"name"`
	Status     string `json:"Status"`
	CreateTime Time   `json:"CreateTime"`
	Labels     map[string]string
}

func ListNamespace(g *gin.Context) {
	ns, err := K8sClient.CoreV1().Namespaces().List(context.Background(), metav1.ListOptions{})
	if err != nil {
		g.Error(err)
		return
	}
	ret := make([]*Namespace, 0)
	for _, item := range ns.Items {
		ret = append(ret, &Namespace{
			Name:       item.Name,
			Status:     string(item.Status.Phase),
			CreateTime: Time(item.CreationTimestamp),
			Labels:     item.Labels,
		})
	}
	//g.JSON(200, ns)      // 可直接输出，但输出的内容数据较多
	g.JSON(200, ret) // 部分数据输出
}
```

点击`CreationTimestamp`进入到源码中可查看起类型：

![image.png](assets/Client-go-7.png)

同理status（这里偷了个懒......上面直接搞了一个string）

![image.png](assets/Client-go-8.png)

同理labels

![image.png](assets/Client-go-9.png)

浏览器访问：[http://127.0.0.1:8080/ns](http://127.0.0.1:8080/ns)

![image.png](assets/Client-go-10.png)

2.  pod

src/service/Pod.go

```go
package service

import (
	"context"
	"github.com/gin-gonic/gin"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	. "k8s/src/lib"
)

type Pod struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
	Nodename  string
	Nodeip    string
	Podip     string
	Status    string
    Labels     map[string]string
}

func ListPod(g *gin.Context) {
	ns := g.Query("namespace")
	podlist, err := K8sClient.CoreV1().Pods(ns).List(context.Background(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	ret := make([]*Pod, 0)
	for _, item := range podlist.Items {
		ret = append(ret, &Pod{
			Name:      item.Name,
			Namespace: item.Namespace,
			Nodename:  item.Spec.NodeName,
			Nodeip:    item.Status.HostIP,
			Podip:     item.Status.PodIP,
			Status:    string(item.Status.Phase),
            Labels:    item.Labels
		})
	}
	//g.JSON(200, podlist)
	g.JSON(200, ret)
}

```

浏览器访问：[http://127.0.0.1:8080/pod?namepsace=kube-system](http://127.0.0.1:8080/pod?namepsace=kube-system)

![image.png](assets/Client-go-11.png)

3.  **deployment**

src/service/Deployment.go

```go
package service

import (
	"context"
	"github.com/gin-gonic/gin"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s/src/lib"
)

type Deployment struct {
	Name                string
	Ready               int32
	UpToDate            int32
	Available           int32
	Replicas            int32
	UnavailableReplicas int32
	Images              string
	Labels              map[string]string
}

func ListDeployment(g *gin.Context) {
	ns := g.Query("namespace")
	dep, err := lib.K8sClient.AppsV1().Deployments(ns).List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		g.Error(err)
		return
	}
	ret := make([]*Deployment, 0)
	for _, item := range dep.Items {
		ret = append(ret, &Deployment{
			Name:                item.Name,
			Ready:               item.Status.ReadyReplicas,
			UpToDate:            item.Status.UpdatedReplicas,
			Available:           item.Status.AvailableReplicas,
			Replicas:            item.Status.Replicas,
			UnavailableReplicas: item.Status.UnavailableReplicas,
			Images:              item.Spec.Template.Spec.Containers[0].Image,
			Labels:              item.Labels,
		})
	}
	//g.JSON(200, dep)
	g.JSON(200, ret)
}
```

浏览器请求：[http://127.0.0.1:8080/dep?namepsace=kube-system](http://127.0.0.1:8080/dep?namepsace=kube-system)

![image.png](assets/Client-go-12.png)

4.  **svc**

src/service/Svc.go

```go
package service

import (
	"context"
	"github.com/gin-gonic/gin"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	. "k8s/src/lib"
)

type Service struct {
	Name       string
	Type       string
	ClusterIp  string
	ExternalIp []string
	Ports      []map[string]interface{}
	Select     map[string]string
}

func ListSvc(g *gin.Context) {
	ns := g.Query("namespace")
	svc, err := K8sClient.CoreV1().Services(ns).List(context.Background(), metav1.ListOptions{})
	if err != nil {
		g.Error(err)
		return
	}
	ret := make([]*Service, 0)
	for _, item := range svc.Items {
		// 先获取到ports相关的数据并进行处理
		// 创建一个切片
		ports := make([]map[string]interface{}, len(item.Spec.Ports))
		for i, port := range item.Spec.Ports {
			ports[i] = map[string]interface{}{
				"name":       port.Name,
				"port":       port.Port,
				"nodePort":   port.NodePort,
				"targetPort": port.TargetPort,
			}
		}
		ret = append(ret, &Service{
			Name:       item.Name,
			Type:       string(item.Spec.Type),
			ClusterIp:  item.Spec.ClusterIP,
			ExternalIp: item.Spec.ExternalIPs,
			Ports:      ports,
			Select:     item.Spec.Selector,
		})

	}
	g.JSON(200, ret)
	return
}
```

浏览器访问：[http://127.0.0.1:8080/svc?namepsace=kube-system](http://127.0.0.1:8080/svc?namepsace=kube-system)

![image.png](assets/Client-go-13.png)

### 进阶

需求：按照deployment为条件显示自己对应pod列表

deployment是通过selector标签去匹的pod的labels，从而识别和管理自己所控制的 Pod。

![image.png](assets/Client-go-14.png)

1.  先编写一个`GetPodByDeployment`**方法**根据namespace和depoyment名称获取对应的pod相关信息

src/service/Deployment.go

```go
package service

import (
	"context"
	"github.com/gin-gonic/gin"
	appsv1 "k8s.io/api/apps/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	. "k8s/src/lib"
)

type pod struct {
	Name       string `json:"name"`
	Namespace  string `json:"namespace"`
	Nodename   string
	Nodeip     string
	Podip      string
	Status     string
	Labels     map[string]string
	CreateTime string
}

type Deployment struct {
	Name                string
	Ready               int32
	UpToDate            int32
	Available           int32
	Replicas            int32
	UnavailableReplicas int32
	Images              string
	Labels              map[string]string
	Pods                []*pod
}

func ListDeployment(g *gin.Context) {
	ns := g.Query("namespace")
	dep, err := K8sClient.AppsV1().Deployments(ns).List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		g.Error(err)
		return
	}
	ret := make([]*Deployment, 0)
	for _, item := range dep.Items {
		ret = append(ret, &Deployment{
			Name:                item.Name,
			Ready:               item.Status.ReadyReplicas,
			UpToDate:            item.Status.UpdatedReplicas,
			Available:           item.Status.AvailableReplicas,
			Replicas:            item.Status.Replicas,
			UnavailableReplicas: item.Status.UnavailableReplicas,
			Images:              item.Spec.Template.Spec.Containers[0].Image,
			Labels:              item.Labels,
		})
	}
	//g.JSON(200, dep)
	g.JSON(200, ret)
}

// 获取deployment关联的pod
func GetPodByDeployment(ns string, dep *appsv1.Deployment) []*pod {
	ctx := context.Background()
	listopt := metav1.ListOptions{
		/*
			Deployment 对象的 Spec.Selector.MatchLabels 数据为键值对map[string]string格式
			例如:
				"app":  "nginx",
				"tier": "frontend",
			而ListOptions.LabelSelector所需要的为字符串格式，需要通过FormatLabelSelector函数转换为字符串
			例如 "key1=value1,key2=value2"。
			metav1 包提供了一些工具来处理标签选择器，避免手动实现。例如：
				metav1.FormatLabelSelector：可以将 LabelSelector 转换为 LabelSelector 字符串。
				metav1.SetAsLabelSelector：可以将 map 转换为 LabelSelector 对象。
		*/
		LabelSelector: metav1.FormatLabelSelector(&metav1.LabelSelector{
			MatchLabels: dep.Spec.Selector.MatchLabels,
		}),
	}
	list, err := K8sClient.CoreV1().Pods(ns).List(ctx, listopt)
	if err != nil {
		panic(err)
	}
	podlist := make([]*pod, len(list.Items))
	for i, item := range list.Items {
		podlist[i] = &pod{
			Name:       item.Name,
			Namespace:  item.Namespace,
			Nodename:   item.Spec.NodeName,
			Nodeip:     item.Status.HostIP,
			Podip:      item.Status.PodIP,
			Status:     string(item.Status.Phase),
			Labels:     item.Labels,
			CreateTime: item.CreationTimestamp.Format("2006-01-02 15:04:05"),
		}
	}
	return podlist
}

// 获取deployment详情
func GetDeployment(g *gin.Context) {
	ns := g.Query("namespace")
	depName := g.Query("name")
	dep, err := K8sClient.AppsV1().Deployments(ns).Get(context.TODO(), depName, metav1.GetOptions{})
	if err != nil {
		g.Error(err)
	}
	ret := make([]*Deployment, 0)
	ret = append(ret, &Deployment{
		Name:                dep.Name,
		Ready:               dep.Status.ReadyReplicas,
		UpToDate:            dep.Status.UpdatedReplicas,
		Available:           dep.Status.AvailableReplicas,
		Replicas:            dep.Status.Replicas,
		UnavailableReplicas: dep.Status.UnavailableReplicas,
		Images:              dep.Spec.Template.Spec.Containers[0].Image,
		Pods:                GetPodByDeployment(ns, dep), // 获取pod
	})
	g.JSON(200, ret)
	return
}
```

main.go

```go
package main

import (
	"github.com/gin-gonic/gin"
	"k8s/src/service"
)

func main() {
	g := gin.Default()
	g.GET("/ns", service.ListNamespace)
	g.GET("/dep", service.ListDeployment)
	g.GET("/pod", service.ListPod)
	g.GET("/svc", service.ListSvc)
	g.GET("/depdetail", service.GetDeployment)
	g.Run(":8080")

}
```

​  

浏览器访问：[http://127.0.0.1:8080/depdetail?namespace=kube-system&name=coredns](http://127.0.0.1:8080/depdetail?namespace=kube-system&name=coredns)

![image.png](assets/Client-go-15.png)

# informer

## 简介

组件架构

![image.png](assets/Client-go-16.png)

![image.png](assets/Client-go-17.png)

![image.png](assets/Client-go-18.png)

  

Informer 负责与 Kubernetes APIServer 进行 Watch 操作，Watch 的资源，可以是 Kubenetes 内置资源对象，也可以是CRD。

Informer 机制的核心工作原理主要包括以下几个步骤：

1.  **List-Watch 机制：** Informer 使用 Kubernetes API 的 List-Watch 机制来获取资源对象的初始列表，并通过 Watch 机制实时接收对象的变化事件。
2.  **SharedInformerFactory：** SharedInformerFactory 是 Informer 的工厂，负责创建和管理多个 SharedInformer。每个 SharedInformer 监听一种资源对象的变化。
3.  **Event Handlers：** 开发者可以注册事件处理器（Event Handlers），在资源对象发生变化时触发相应的处理逻辑。Event Handlers 是 Informer 的核心扩展点，用于实现自定义的业务逻辑。
4.  **Delta FIFO Queue：** 通过 Delta FIFO Queue，Informer 在收到资源对象的变化事件时，将事件推送到队列中。Event Handlers 从队列中取出事件进行处理。
5.  **Resync：** Informer 支持定期的全量同步（Resync）机制，以确保本地缓存与实际状态的一致性。定期地对资源对象进行全量同步，更新本地缓存。

Informer 是一个带有本地缓存以及索引机制的核心工具包，当请求为查询操作的时候优先从本地缓存内去查找数据，而 创建、更新、删除，这类的操作，则会根据事件通知写入到队列 DeltaFIFO 中，同时对应的事件处理过后，更新本地缓存，使本地缓存与 ETCD的数据保持一致。

Informer 抽象出来的这个缓存层，将查询相关操作的压力接收了下来这样就不必每次去调用APIServer的接口，减轻了APIServer 的数据交互压力从上图就能看出来，Informer 是有多个组件构成的：

-   **Reflector**：使用 List-Watch 来保证本地缓存数据的，准确性、顺序性和一致性的List 对应资源的全量列表数据，Watch 负责变化部分的数据，Watch 指定的 Kubernetes 资源，当 watch 的资源发生变时，触发变更的事件，比如Added，Updated 和 Deleted 事件，并将资源对象的变化事件存放到本地队列 DeltaFIFo中。
-   **DeltalFIFO**：是一个增量队列，记录了资源变化的过程，Reflector 就相当于队列的生产者。这个组件可以拆分为两部分来理解，FIFO 就是一个队列，拥有队列基本方法，例如 ADD，UPDATE，DELETELIST，POP，CLOSE等，Delta 是一个资源对象存储，保存存储对象的消费类型比如 Added，Updated，Deleted等。
-   **Indexer**：用来存储资源对象并自带索引功能的本地存储，Reflector 从 DeltaFIFO将消费出来的资源对象存储到lndexer，Indexer 与 Etcd 中的数据完全保持一致。从而 client-go 可以本地读取，减少Kubernetes APIServer 的数据交互压力。

## list-watch

### 简介

List-Watch 机制是 Kubernetes 中的异步消息通知性机制，通过它能有效的确保了消息的实时性、顺序性和可靠。

List-Watch，它是分为两部分：

-   List 负责调用资源对应的 Kubernetes APIServer的 RESTfulAPI 获取全局数据列表，并同步到本地缓存中。
-   Watch 负责监听资源的变化，并调用相应事生的处理函数进行处理，同时更新本地缓存，使本地缓存与Etcd 中数据，保持一致。

List 是基于HTTP 中的短链接实现。watch 则是基于HTTP 长链接实现，使用长链接的方式，极大的减轻了 Kubernetes APIServer 的访问压力。

  

client-go中的`k8s.io/client-go/tools/cache`包`informer`对象对list-watch机制进行了封装

-   初始化调用List api获得全量list 缓存（本地缓存）
-   调用watch api watch资源，当资源发生变更通过一定机制维护缓存，减少访问apiserver的压力

**文章资料：**

-   [Client-go源码分析之Reflector](https://bbs.huaweicloud.com/blogs/332737?utm_source=luntan&utm_medium=bbs-ex&utm_campaign=other&utm_content=content)
-   华为云不错的视频：[list-watch机制原理详解](https://education.huaweicloud.com/courses/course-v1:HuaweiX+CBUCNXI042+Self-paced/courseware/6817c598390d4a008e5c6f45777aa10b/2c6b9bb49c75422b9691c7cd87d59669/)，[client-go(kubernetes)的ListerWatcher解析](https://blog.csdn.net/weixin_42663840/article/details/114379569)。
-   [https://www.bilibili.com/video/BV1na411U71j?spm\_id\_from=333.788.player.switch&vd\_source=754a7ea17285a3199d3118f9560550bf](https://www.bilibili.com/video/BV1na411U71j?spm_id_from=333.788.player.switch&vd_source=754a7ea17285a3199d3118f9560550bf)

### watch初试

以deployment简单例子的开始

/src/service/demo.go

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	appsv1 "k8s.io/api/apps/v1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/retry"
    "k8s.io/client-go/util/homedir"
	"os"
	"path/filepath"
)

func main() {
	var kubeconfig *string
	// client-go包提供的方法，获取家目录
	if home := homedir.HomeDir(); home != "" {
        // 拼接取config文件：~/.kube/config
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file)")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "a string")
	}

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

    // 创建监听对象：监听default命名空间下的deployment的变化
    w, err := clientset.Appsv1().deployments("default").Watch(context.TODO(), metav1.ListOptions{})
    if err != nil {
		panic(err)
	}
    fmt.Println("start......")
    for {
        seletct {
            // 从chan中获取数据，返回的事件实例存在两个属性：Type，Object
            case e, _ := <-w.ResultChan():
                fmt.Println(e.Type, e.Object)
                // e.Type: 表示事件变化的类型，ADDEN、DELETE、MODIFIED等操作
                // e.Object: 表示变化后最新的信息数据
        }
    }
}
```

/src/service/informerTest.go

```go
// 新版client-go(v6.0.0 及之后) 中NewInformer()被标记为Deprecated（反对使用），而是使用功能更强性能更好的NewSharedInformerFactory() 或 NewSharedInformerFactoryWithOptions() 代替
cache.NewInformer()
```

## Reflector

Refector是 Cient-Go中，用来监听指定资源的组件，当资源发生变化的时候，例如 增加、更新、删除等操作的时候，会以事件的形式存入本地队列，然后有对应的方法处理。

在Reflector 中，核心的部分就是 List-watch，其他功能基本上也是围绕这它来搞的

在实例化Reflector的过程中，有一个ListWatcher的接口对象，这个结构对象有两个方法：List和Watch，这两个方法就实现了List-Watch功能。

下面咱们来说说 Reflector 的核心逻辑，其实它的核心逻辑可以简单归为三部分。

-   **List：**调用 List 方法获取资源全部列表数据，转换为资源对象列表，然后保存到本地缓存中。
-   **定时同步**：定时器定时触发同步机制，定时更新缓存数据，在 Reflector 的结构体对象中，是可以配置定时同步的周期时间的。
-   **Watch**：监听资源的变化，并且调用对应的事件处理函数来进行处理。

Reflector 组件对于数据更新同步，都是基于 ResourceVersion 来进行的，每个资源对象都会有 Resourceversion 这个属性，当数据发生变化的时候ResourceVersion 也将会以递增的形式更新，这样就确保了变更事件的顺序性。

Watch 资源对象的时候，ResourceVersion 可以根据我们的需求，进行配置。

-   ResourceVersion 未设置的时候，从最新版本开始监听。

-   为了建立初始状态，Watch 从起始资源版本中存在的所有资源实例的合成"添加"事件开始。以下所有监视事件都针对在 Watch 开始的资源版本之后发生的所有更改。

-   ResourceVersion 设置为“0”的时候，则表示从任意版本开始监听。

-   以这种方式初始化的监视可能会返回任意陈旧的数据。首选可用的最新资源版本，但不是必繁的。允许任何起始资源版本。由分区或过时的缓存，Watch 可能从客户端之前观察到的更旧的资源版本开始，特别是在高可用性配置中。不能容忍这种明显倒带的客户不应该用这种语义启动 Watch。
-   为了建立初始状态，Watch 从起始资源版本中存在的所有资源实例的合成"添加"事件开始。以下所有监视事件都针对在 watch 开始的资源版本之后发生的所有更改。

-   ResourceVersion 从指定版本开始监听

-   监视事件适用于提供的资源版本之后的所有更改。Watch 不会以所提供资源版本的合成"`ADDED`(添加)"事件启动。由于客户端提供了资源版本，因此假定客户端已经具有起始资源版本的初始状态。

ResourceVersion 不仅是在 Reflector 中有重要的应用，Update 机制或 Patch 机制也是会基于 Resourceversion 来比较两个资源对象，确定是否有变化的。

当Watch 资源断开的时候，Reflector 会重新进行 List-Watch 以确保数据的可靠性。

同时 Watch 使用 HTTP 的长链接的形式进行资源的监听，保证了数据实时性的同时，还减轻 ubernetes APIServer 的访问压力。

## DeltaFIFO

DeltaFIFO 是一个增量的本地队列，记录了资源对象的变化过程。

它的生产者就是 Reflector 组件。将监听的对象，同步到 DeltaFIFo中。DeltaFIFO 是分为两部分的。分别 FIFO和 DeltaFIFO

-   FIFO就是一个先入先出的本地队列，负责接收 Reflector 传递过来的事件，并将其按照顺序存储，然后等待事件的处理函数进行处理，同时若出现多个相同的事件，则只会被处理一次。
-   Delta 则是资源对象的变化，例如增加、删除、修改等。

FIFO 既然是一个队列那么就肯定有队列相关的操作方法，在这里就是通过 Queue 这个接口对象，来实现队列所需的方法的，同时还根据需求拓展了一些其他的方法。例如: Pop、AddlfNotPresent 等。

Delta 有两个属性，分别是 Type 和 Object。

-   Type 就表示这个事件的类型，就比如说 Added 表示增加
-   Updated 表示更新等等Obiect是一个interface 类型的，它就表示一个具体 Kubernetes 资源对象，例如: Pod、Service等

## Indexer

通过字面意思我们可以看出它叫做“索引器”对吧

它其实就是Informer 中 LocalStore 的部分。Indexer 本身是一个存储，同时在存储的基础上扩展了索引的功能。

Reflector 通过 DeltaFIFO 一系列的操作后，然后更新存储到 Indexer 中。

indexer 中的数据，与 Etcd 中的数据是完全一致的，当 CIient-Go 需要获取数据的时候，则无须每次都从 APIServer 中获取、从而减轻了 APIServer 的请求力在更深入了解Indexer 之前，我们还需要知道Indexer 中几个非常重要的概念

-   IndexFunc：索器函数、用于计算一个资源对象的索值列表，可以根据需求定义其他的，比如根据 Label 标签、Annotation 等属性来生成索值列表。
-   Index：存储数据，要查找某个命名空间下面的 Pod，那就要让 Pod 按照其命名空间进行索，对应的 ndex 类型就是 map[namespace].sets.pod。。
-   Indexers：存储索引器，key为索引器名称，value 为索引器的实现函数，例如: ma["namespace"]MetaNamespacelndexFunc。
-   Indices：存储缓存器，key 为索引器名称，value 为缓存的数据，例如: map["namespace"]mapnamespace.sets.pod。

最容易混淆的是 ndexers 和 ndices 这两个概念，我们可以这样理解: ndexers 是存储索引(生成索键)的，ndices 里面是存储的真的数据(对象键)

**ThreadSafeMap**

这是一个并发安全的存储，Indexer 就是在 ThreadSafeMap 基础上进行封装的，实现了索引相关的功能Cache

```go
k8s.io/client-go/tools/cache/thread_safe_store.go
```

**Cache**

Cache是对 ThreadSafeStore 的一个再次封装，很多操作都是直接调用的 ThreadSafeStore 的操作实现的。

```go
k8s.io/client-go/tools/cache/store.go
```

# SharedInformer

## 介绍

前面介绍过Informer 会将资源缓存在本地以供自己后续使用。但 Kubernetes 中运行了很多控制器，有很多资源需要管理，难免会出现一个资源受到多个控制器管理。为了应对这种场景，可以通过 `Sharedlnformer` 来创建一份供多个控制器共享的缓存，同时也减少了系统的内存开销。使用了`Sharedlnformer` 之后，不管有多少个控制器同时读取事件，Sharedlnformer 只会调用一个Watch API来 watch 上游的API Server，大大降低了API Server 的负载。实际上 kube-controller-manager就是这么工作的。

Sharedlnformer 是一个接口，包含添加事件，当有资源变化时，会回调通知使用者，启云函数及获取是否全量对象是否已经同步到本地存储中。一般是使用 `SharedinformerFactory` 来管理控制器需要的资源对象的infrmer 实例，使用map的结构进行存储资源对象的Informer。

**Sharedlndexlnformer:** SharedIndexlnformer 在 Sharedlnformer 基础上扩展了添加和获取Indexers 的能力

**SharedlnformerFactory:** SharedInformerFactory 为所有已知的GVR 提供共享的 informer。

示例：

-   **通过indexer查询同步到本地的资源数据：**

```go
package main

import (
	"flag"
	"fmt"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"path/filepath"
	"time"
)

func main() {
	var kubeconfig *string
	// 获取kubeconfig文件路径
	//if home := homedir.HomeDir(); home != "" {
	//	kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")
	//} else {
	//	kubeconfig = flag.String("kubeconfig", "", "a string")
	//}
	//flag.Parse()
	kubeconfig = flag.String("kubeconfig", filepath.Join("kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	// 初始化SharedInformerFactory，传入客户端及同步数据的周期
	shareInformerfactory := informers.NewSharedInformerFactory(clientset, 0)

	// 查询pod的数据，生成podInformer对象
	podInformer := shareInformerfactory.Core().V1().Pods()

    // 注册事件处理程序
    
	// 生成一个indexer，便于数据查询
	indexer := podInformer.Lister()

	// 启动Informer
	shareInformerfactory.Start(nil)

	// 等待数据同步完成
	shareInformerfactory.WaitForCacheSync(nil)

	// 三秒中查询一次数据， 此时如果修改pod的数据，会实时输出
	for _ = range time.Tick(time.Second * 3) {
        // 利用indexer查询数据，每次查询时重新获取最新的 Pod 列表
    	pods, _ := indexer.List(labels.Everything())
		for _, pod := range pods {
			fmt.Printf("name: %v, namespace: %v \n", pod.Name, pod.Namespace)
		}
	}
    // 阻塞，不让程序退出
    select {}
}
```

-   **pod、deployment添加事件监听**

```go
package main

import (
	"flag"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"path/filepath"
	"time"
)

type PodHandler struct {
}

func (p *PodHandler) OnAdd(obj interface{}, isInInitialList bool) {
	if pod, ok := obj.(*corev1.Pod); ok {
		fmt.Printf("name: %v, namespace: %v  is added!\n", pod.Name, pod.Namespace)
	}
}
func (p *PodHandler) OnUpdate(oldObj, newObj interface{}) {
	if pod, ok := newObj.(*corev1.Pod); ok {
		fmt.Printf("name: %v, namespace: %v  is changed!\n", pod.Name, pod.Namespace)
	}
}
func (p *PodHandler) OnDelete(obj interface{}) {
	if pod, ok := obj.(*corev1.Pod); ok {
		fmt.Printf("name: %v, namespace: %v  is deleted!\n", pod.Name, pod.Namespace)
	}
}

func main() {
	var kubeconfig *string
	// 获取kubeconfig文件路径
	//if home := homedir.HomeDir(); home != "" {
	//	kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")
	//} else {
	//	kubeconfig = flag.String("kubeconfig", "", "a string")
	//}
	//flag.Parse()
	kubeconfig = flag.String("kubeconfig", filepath.Join("kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	// 初始化SharedInformerFactory，传入客户端及同步数据的周期
	shareInformerfactory := informers.NewSharedInformerFactory(clientset, 0)

	// 查询pod的数据，生成podInformer对象
	podInformer := shareInformerfactory.Core().V1().Pods()

	// 注册事件处理程序
	podInformer.Informer().AddEventHandler(&PodHandler{})

	// 启动Informer
	shareInformerfactory.Start(wait.NeverStop)

	select {}
}
```

输出：

首次会监听到Onadd加载全部的pod

```go
name: nginx-7c658794b9-js6z9, namespace: default  is added!
name: calico-kube-controllers-595b58b579-5xmkw, namespace: kube-system  is added!
name: calico-node-72hxj, namespace: kube-system  is added!
name: etcd-k8s-master1, namespace: kube-system  is added!
name: coredns-6d8c4cb4d-hg7pm, namespace: kube-system  is added!
name: kube-proxy-hfznn, namespace: kube-system  is added!
name: kube-proxy-pjz5j, namespace: kube-system  is added!
name: coredns-6d8c4cb4d-bvlkq, namespace: kube-system  is added!
name: kube-apiserver-k8s-master1, namespace: kube-system  is added!
name: kube-controller-manager-k8s-master1, namespace: kube-system  is added!
name: calico-node-txq8j, namespace: kube-system  is added!
name: kube-scheduler-k8s-master1, namespace: kube-system  is added!

name: nginx-7c658794b9-rrfhm, namespace: default  is added!
name: nginx-7c658794b9-tq5qk, namespace: default  is added!
name: nginx-7c658794b9-rrfhm, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-rrfhm, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-rrfhm, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-rrfhm, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is changed!
name: nginx-7c658794b9-tq5qk, namespace: default  is deleted!
```

-   **deployment的事件监听**

```go
package main

import (
	"flag"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	appsv1 "k8s.io/api/apps/v1"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"path/filepath"
	"sync"
)

// 考虑并发原因使用sync.Map
type DepMap struct {
	data sync.Map
}

type DepHandler struct {
}

func (d *DepHandler) OnAdd(obj interface{}, isInInitialList bool){
	fmt.Printf("name: %s, namespace: %s",obj.(*appsv1.Deployment).Name,obj.(*appsv1.Deployment).Namespace)
}

func (d *DepHandler) OnUpdate(oldObj, Newobj interface{}) {
	if dep, ok := Newobj.(*appsv1.Deployment); ok{
		fmt.Printf("Name: %s, Namespace: %s", dep.Name, dep.Namespace)
	}
}

func (d *DepHandler) OnDelete(obj interface{}) {
	fmt.Printf("delete: %s",obj.(*appsv1.Deployment).Name)
}

func main() {
	var kubeconfig *string
	// 获取kubeconfig文件路径
	//if home := homedir.HomeDir(); home != "" {
	//	kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")
	//} else {
	//	kubeconfig = flag.String("kubeconfig", "", "a string")
	//}
	//flag.Parse()
	kubeconfig = flag.String("kubeconfig", filepath.Join("kube", "kubeconfig"), "(optional) absolute path to the kubeconfig file)")

	// 构建配置对象
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	// 创建客户端对象clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	// 初始化SharedInformerFactory，传入客户端及同步数据的周期
	shareInformerfactory := informers.NewSharedInformerFactory(clientset, 0)

	// 查询pod的数据，生成podInformer对象
	depInformer := shareInformerfactory.Apps().V1().Deployments()

	// 生成一个indexer，便于数据查询
	//indexer := podInformer.Lister()

	// 注册事件处理程序
	depInformer.Informer().AddEventHandler(&DepHandler{})

	// 启动Informer
	shareInformerfactory.Start(wait.NeverStop)

	select {}

}
```

## 整合使用

### 简单使用

封装方法：src/lib/informer.go

```go
package lib

import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/informers"
	"time"
)

// 包内全局私有变量
var sharedInformerFactory informers.SharedInformerFactory

func NewSharedInformerFactory(stopCh <-chan struct{}) (err error) {
	// 实例化sharedInformerFactory
	sharedInformerFactory = informers.NewSharedInformerFactory(K8sClient, time.Second*3)

	// 定义需要启动监听资源
	gvrs := []schema.GroupVersionResource{
		{Group: "", Version: "v1", Resource: "pods"},
		{Group: "", Version: "v1", Resource: "services"},
		{Group: "", Version: "v1", Resource: "namespaces"},

		{Group: "apps", Version: "v1", Resource: "deployments"},
		{Group: "apps", Version: "v1", Resource: "statefulsets"},
		{Group: "apps", Version: "v1", Resource: "daemonsets"},
	}
	for _, v := range gvrs {
		// 创建informer
		_, err = sharedInformerFactory.ForResource(v)
		if err != nil {
			return
		}
	}
	// 启动所有创建的 informer，传入stopCh 通道结束标识
	sharedInformerFactory.Start(stopCh)
	// 等待所有的 informer 同步数据完成，传入stopCh 通道结束标识
	sharedInformerFactory.WaitForCacheSync(stopCh)

	return
}

// 实例化方法
func Get() informers.SharedInformerFactory {
	return sharedInformerFactory
}

// 初始化方法
func Setup(stopCh <-chan struct{}) (err error) {
	err = NewSharedInformerFactory(stopCh)
	if err != nil {
		return
	}
	return
}
```
```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/labels"
	"k8s/src/lib"
)

func main() {
	stopCh := make(chan struct{})

	//初始化 informer
	err := lib.Setup(stopCh)
	if err != nil {
		panic(err)
	}
    // 通过本地缓存获取资源数据
	items, err := lib.Get().Core().V1().Pods().Lister().List(labels.Everything())
	if err != nil {
		panic(err)
	}
	for _, item := range items {
		fmt.Printf("name: %s, namespace: %s \n", item.Name, item.Namespace)
	}
}
```

### 结合gin

注意：当前环境依赖：简单结合gin---》准备工作---》1、只封装ClientSet，所封装的方法

```go
package main

import (
	"github.com/gin-gonic/gin"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/util/proxy"
	"k8s.io/client-go/rest"
	"k8s/src/lib"
	"net/http"
)

func main() {
	stopCh := make(chan struct{})

	//初始化 informer
	err := lib.Setup(stopCh)
	if err != nil {
		panic(err)
	}
	items, err := lib.Get().Core().V1().Pods().Lister().List(labels.Everything())
	if err != nil {
		panic(err)
	}
	g := gin.Default()
	// 获取固定资源列表
	g.GET("/pod/list", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"code": 200,
			"msg":  "ok",
			"data": items,
		})
	})

	// 根据传入URL路径参数动态获取对应资源数据
	g.GET("/:resource/:group/:version", func(c *gin.Context) {
		resource := c.Param("resource")
		group := c.Param("group")
		version := c.Param("version")
		// 组合成gvr
		gvr := schema.GroupVersionResource{
			Group:    group,
			Version:  version,
			Resource: resource,
		}
		// 通过gvr获取到对应资源的informer
		i, err := lib.Get().ForResource(gvr)
		if err != nil {
			panic(err.Error())
		}
		items, err := i.Lister().List(labels.Everything())

		c.JSON(http.StatusOK, gin.H{
			"code": 200,
			"msg":  "ok",
			"data": items,
		})
	})

	// 转发请求到APIServer
	g.Any("/apis/*action", func(c *gin.Context) {
		// transport: client-go 对http.client做的一个封装
        // 需要传入config对象，这里已经在前面有所封装：简单结合gin---》准备工作---》1、只封装ClientSet
		t, err := rest.TransportFor(lib.Config)
		if err != nil {
			panic(err)
		}
		// 配置请求地址
		s := *c.Request.URL
		s.Host = "192.168.9.20:6443"
		s.Scheme = "https"

		httpProxy := proxy.NewUpgradeAwareHandler(&s, t, false, false, nil)
		httpProxy.UpgradeTransport = proxy.NewUpgradeRequestRoundTripper(t, t)
		httpProxy.ServeHTTP(c.Writer, c.Request)
	})

	g.Run(":8888")
}
```

请求接口：

/pod/list

![image.png](assets/Client-go-19.png)

**根据传入URL路径参数动态获取数据**

-   **pod**

![image.png](assets/Client-go-20.png)

​  

-   **deployment**

![image.png](assets/Client-go-21.png)

​  

-   **daemonset**

![image.png](assets/Client-go-22.png)

**直接转发到APIServer的**

-   **deployment**

![image.png](assets/Client-go-23.png)

-   **daemonset**

![image.png](assets/Client-go-24.png)

还能是其他apps资源组的资源：statefulset等

# 参考资料

[https://www.fdevops.com/2022/06/26/31114](https://www.fdevops.com/2022/06/26/31114)

[https://www.yuque.com/duiniwukenaihe/hg6ymd/vh8s8p](https://www.yuque.com/duiniwukenaihe/hg6ymd/vh8s8p)

[https://www.yuque.com/coolops/golang/cseuca](https://www.yuque.com/coolops/golang/cseuca)
