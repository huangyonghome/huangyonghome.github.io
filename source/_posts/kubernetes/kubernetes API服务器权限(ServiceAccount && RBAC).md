---
title: kubernetes API服务器权限(ServiceAccount && RBAC)
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## kubernetes API服务器权限(ServiceAccount && RBAC)

### ServiceAccount介绍

​		每个Pod都与一个ServiceAccount相关联,它代表了运行在pod中应用程序的身份证明.每个pod在启动的时候kubernetes会自动挂载ServiceAccount的TOKEN到pod容器的`/var/run/secretes/kubernetes.io/serviceaccount/token`文件.pod内的应用程序通过这个token连接API服务器.API服务器的身份验证插件会对ServiceAccount进行身份认证.ServiceAccount所拥有的权限决定了pod内的应用程序所尝试执行的操作是否被允许

​		ServiceAccount只不过是一种运行在pod中的应用程序和API服务器身份认证的一种方式.应用程序通过在请求中传递serviceaccount的token和API服务器通信

---

### 了解ServiceAccount资源

​		ServiceAccount和pod,Secret,ConfigMap等一样,本身也是一种资源.他们作用在单独的命名空间.kubernetes为每个命名空间自动创建一个默认的ServiceAccount(名字是default),可以像查看其它资源一样使用`kubectl get sa`来查看ServiceAccount列表

<!--more-->

```
[root@k8s-master ~]# kubectl get sa
NAME      SECRETS   AGE
default   1         51d
```

* 同一个命名空间下的多个pod可以使用同一个ServiceAccount
* pod只能使用同一个命名空间下的ServiceAccount
* 如果在pod的manifest文件中没有显示的指定ServiceAccount名称.则默认使用default ServiceAccount



### 创建ServiceAccount

​		为什么要去创建新的ServiceAccount,而不是所有pod都使用默认的ServiceAccount? 为了集群的安全性,尽量做到权限最小分配的原则,只对有需要的Pod配置有相应较高权限的ServiceAccount,而其他pod使用default默认ServiceAccount应该不允许他们检索或者修改部署在集群中的任何资源.

​		下面实验一下如何创建其他ServiceAccount,并且分配给Pod

* 创建ServiceAccount很简单,直接使用`kubectl create serviceaccount`命令

```
[root@k8s-master ~]# kubectl create serviceaccount foo
serviceaccount/foo created

[root@k8s-master ~]# kubectl describe sa foo
Name:                foo
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   foo-token-ktjj5
Tokens:              foo-token-ktjj5
Events:              <none>
[root@k8s-master ~]#
```

上面就创建了一个名为foo的serviceaccount.如果查看这个名为foo-token-ktjj5的TOKEN秘钥可以发现这个ServiceAccount也和default默认的ServiceAccount一样具有相同的条目(CA证书,命名空间,token),当然这2个token本身是不一样的

```
[root@k8s-master ~]# kubectl describe secret foo-token-ktjj5
Name:         foo-token-ktjj5
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: foo
              kubernetes.io/service-account.uid: 28cb7741-eaf3-46bb-bd58-3f1ddb748c2e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InI3cE50azROV1pseDJqNmxVQTlSWjF6Vk9YVFFnamZEWFF5RG56YTJhaWMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImZvby10b2tlbi1rdGpqNSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJmb28iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIyOGNiNzc0MS1lYWYzLTQ2YmItYmQ1OC0zZjFkZGI3NDhjMmUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpmb28ifQ.KdfyYVkb7Ye_sV8yUWqU6I8fe8Pck7cyTDdwh08Y9w0f4cwJKqIwsuVn39VCpYIH3PaDsKoxBv7Bpahn6lfiP3MgA4zcWlX4vtBUxJCAt-GBXBTcHkqQ6BwtxFzkY4rgsXLd5HuqsrqrZbxrSM1zfNovPccRgklN3kbz0BuxTmKQ65E9DFhAt5kmzajO3qvAB_ymGeoJLMul1fZEqLnV48UEKN5KFAxVfwo0b4On2LcKOVkmG_P7yO7X6TsE6zEN03kvoxjOd0vnJrGUCT9fu0KlabRyDsyDWsua-Su2kv-gDg3O6zQxqDGgEwRxTb2-7Cbv6PaFbfCwSRnCC8Y6FA
```

* 将ServiceAccount分配给pod

将ServiceAccount赋值给Pod非常简单,进需要在pod的manifest文件中的spec字段下指定serviceAccountName即可.例如下面之前学习过的curl镜像和kubectl-proxy的ambassador镜像

```
apiVersion: v1
kind: Pod
metadata:
  name: curl-custome-sa
spec:
  #指定pod使用哪个serviceaccount
  serviceAccountName: foo
  containers:
    - name: main
      image: tutum/curl
      command: ['sleep','9999999']
    - name: ambassador
      image: luksa/kubectl-proxy:1.6.2
```

> ServiceAccount必须在pod创建之前新建,后续不能被修改

* 使用新创建的ServiceAccount访问API服务器.(通过ambassador容器)

```
[root@k8s-master ~]# kubectl exec -it curl-custome-sa -c main curl localhost:8001/api/v1/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:default:foo\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
```

还是无法访问? 这是因为RBAC(基于角色权限控制)插件没有给这个ServiceAccount进行权限授权.

---

### 介绍RBAC

​		在1.6版本开始,集群安全性显著提高.RBAC在集群中默认开启.RBAC会阻止未授权的用户查看和修改集群状态.API服务器对外暴露了REST接口,用户可以通过向服务器发送HTTP请求来执行动作.

​		REST客户端发送GET,POST,PUT,DELETE和其他类型的HTTP请求到特定的URL资源.这些资源可以是kubernetes的Pod,Service,Secret等等.

​		RBAC支持的支持一些动词(例如,get,create,update)等映射到客户端请求的HTTP方法(GET,POST,PUT).完整的映射方法如下表

| HTTP方法 |        RBAC动词        |
| :------: | :--------------------: |
| GET,HEAD | get(或者watch用于监听) |
|   POST   |         create         |
|   PUT    |         update         |
|  PATCH   |         patch          |
|  DELETE  |         delete         |

​		除了可以对全部资源类型应用安装权限,RBAC规则还可以引用于特定的资源实例,以及非资源URL路径.(并不是API服务器对外暴露的每个路径都映射到一个资源),例如/api路径,/healthz健康路径

​		RBAC授权插件将用户角色作为决定用户能否执行操作的关键因素主体.(可以是一个人,一个ServiceAccount,或者一组人,一组ServiceAccount).一个用户可以有多个角色.

---

### 介绍RBAC资源

RBAC授权规则通过四种资源来进行配置.可以分为两个组:

* Role(角色)和ClusterRole(集群角色).他们指定了在资源上可以执行哪些动作
* RoleBinding(角色绑定)和ClusterRoleBinding.(集群角色绑定).他们将上述角色绑定到特定的用户,组或者ServiceAccount上

> 角色定义了可以有什么访问权限,而角色绑定决定了谁可以做这些操作

角色和集群角色,或者角色绑定和集群角色绑定之间的区别在于角色和角色绑定是命名空间资源.而集群角色和集群角色绑定是集群级别的资源(独立资源,不属于任何命名空间).

* 单个命名空间可以创建多个角色,以及多个角色绑定
* 尽管角色绑定是在命名空间下的.但是也可以引用其他命名空间下的集群角色.

---

### 演示RBAC的实践

* ##### 创建2个命名空间,然后在2个命名空间下运行`kubectl-proxy` pod.

下面这个配置创建了`foo`,`bar`这2个名称空间.以及在每个名称空间下创建了一个`kubectl-proxy`的test容器

```bash
[root@k8s-master ~]# kubectl create ns foo
namespace/foo created
[root@k8s-master ~]# kubectl create ns bar
namespace/bar created

[root@k8s-master ~]# kubectl run test --image=luksa/kubectl-proxy -n foo
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/test created

[root@k8s-master ~]# kubectl run test --image=luksa/kubectl-proxy -n bar
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/test created
```

列出各名称空间下的pod

```bash
[root@k8s-master ~]# kubectl get po -n foo
NAME                    READY   STATUS    RESTARTS   AGE
test-7b4bb6b9ff-qpppf   1/1     Running   0          28m

[root@k8s-master ~]# kubectl get po -n bar
NAME                    READY   STATUS    RESTARTS   AGE
test-7b4bb6b9ff-jl6v5   1/1     Running   0          42m
```

此时由于还没有授权,进入pod容器内部,对API服务器发起http访问请求时.API服务器不允许Pod访问services服务

```

[root@k8s-master ~]# kubectl exec -it test-7b4bb6b9ff-qpppf -n foo sh

/ # curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"services\" in API group \"\" in the namespace \"foo\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
}/ #
```

---

* ##### 使用Role和RoleBinding

  ##### 创建Role

  Role资源定义了哪些操作可以在哪些资源上执行.下面的配置文件定义了一个Role.允许用户获取并列出foo命名空间中的服务

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  #该role所在的命名空间,如果没有填写,则此role应用在默认的命名空间
  namespace: foo
  name: service-reader
#以下是此role定义的规则
rules:
    #Service是核心apiGroup的资源,所以没有apiGroup名,就是""空
  - apiGroups: [""]
    #该Role角色授权的规则,允许get,list访问权限
    verbs: ["get","list"]
   #这个规则和服务有关,(必须使用复数)
    resources: ["services"]
~
```

Role配置文件中定义了以下信息:

* Role所处的名称空间
* Role角色名
* verbs: 角色允许的访问权限
* resources: 角色允许访问的资源

创建role

```bash
[root@k8s-master ~]# kubectl create -f serviceaccount-service-reader.yaml
role.rbac.authorization.k8s.io/service-reader created

[root@k8s-master ~]# kubectl get role -n foo
NAME             AGE
service-reader   6m18s

[root@k8s-master ~]# kubectl describe role -n foo service-reader
Name:         service-reader
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  services   []                 []              [get list]
[root@k8s-master ~]#
```

* ##### 将Role角色绑定到ServiceAccount

  ​      Role角色定义了哪些操作可以执行,但是没有指定谁拥有该角色.要做到这一点,需要将角色绑定到一个主体上.它可以是一个usr(用户),或者一个ServiceAccount.或者一个组.

  ​		通过创建一个RoleBinding资源来实现将角色绑定到主体.使用`kubectl create rolebinding`.

  完整格式是`kubectl create rolebinding 绑定名 --role=角色名 --serviceaccount=命名空间:ServiceAccount名 -n 命名空间`.下面是具体的例子实现

  ```bash
  [root@k8s-master ~]# kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo
  rolebinding.rbac.authorization.k8s.io/test created
  
  [root@k8s-master ~]# kubectl get rolebinding test -n foo  -o yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    creationTimestamp: "2020-05-05T09:45:58Z"
    name: test
    namespace: foo
    resourceVersion: "10719596"
    selfLink: /apis/rbac.authorization.k8s.io/v1/namespaces/foo/rolebindings/test
    uid: a400ece9-ceb7-4a57-8687-f385a69b9190
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: service-reader
  subjects:
  - kind: ServiceAccount
    name: default
    namespace: foo
  ```

  ​		在上面的例子中创建了一个test的rolebinding的角色绑定资源.将`service-reader`这个Role角色绑定到foo名称空间下的default默认ServiceAccount下.

  ​		通过yaml文件显示,RoleBinding角色绑定引用了单一的角色(从`roleRef`字段可以看到引用的Role角色信息).RoleBinding可以将单个角色绑定到多个主体.

  ​		因为这个RoleBinding将service-reader这个Role角色绑定到了foo命名空间下的ServiceAccount下.所以现在foo下的pod可以访问集群的Service资源了

  ```
  #进入foo命名空间下的Pod容器内部,通过API服务器访问services
  / # curl localhost:8001/api/v1/namespaces/foo/services
  {
    "kind": "ServiceList",
    "apiVersion": "v1",
    "metadata": {
      "selfLink": "/api/v1/namespaces/foo/services",
      "resourceVersion": "10720460"
    },
    "items": []  #items列表为空很正常,因为foo名称空间下没有任何Serivce资源
  }/ #
  ```

* ##### 在RoleBinding角色绑定中使用其他名称空间的ServiceAccount

  ​	刚才在foo的名称空间下创建了Role和RoleBinding.并且foo下的pod可以正常通过API服务器访问Service资源了.但是很显然bar名称空间下pod没有任何访问权限.

  ​	可以修改foo名称空间中的RoleBinding,并添加另一个pod的ServiceAccount.即使这个ServiceAccount在另一个不同的名称空间中.

  ​		修改RoleBinding资源名为test的配置文件

```
[root@k8s-master ~]# kubectl edit rolebinding test -n foo
rolebinding.rbac.authorization.k8s.io/test edited

#在配置文件中最后新增一个subjects配置.引用来自bar名称空间中的default ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2020-05-05T09:45:58Z"
  name: test
  namespace: foo
  resourceVersion: "10719596"
  selfLink: /apis/rbac.authorization.k8s.io/v1/namespaces/foo/rolebindings/test
  uid: a400ece9-ceb7-4a57-8687-f385a69b9190
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: service-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
- kind: ServiceAccount
  name: default
  namespace: bar
```

此时,bar名称空间下的Pod可以访问foo名称空间下的Service资源(但是仍然不能访问bar自己名称空间下的资源)

```bash
/ # curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/foo/services",
    "resourceVersion": "10722489"
  },
  "items": []
  
  
/ # curl localhost:8001/api/v1/namespaces/bar/services
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:bar:default\" cannot list resource \"services\" in API group \"\" in the namespace \"bar\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
```

---

### ClusterRole和ClusterRoleBinding

​	Role和RoleBinding都是名称空间的资源,他们属于一个单一的名称空间资源上.但是如上面那个例子,RoleBinding可以引用来自其他名称空间中的ServiceAccount.

​	除了这些名称空间里的资源,还存在2个集群级别的RBAC资源:**ClusterRole**和**ClusterRoleBinding**.之所以存在集群级别的 资源是因为常规的**Role**和**RoleBinding**只允许访问和角色在同一个名称空间中的资源.如果希望允许不同名称空间下的Pod可以互相访问资源,则需要在每个名称空间中创建一个相同的**Role**和**RoleBinding**.

​	另外,一些特定的资源完全不在任何名称空间中(例如.Node,PV,namespace等等).我们也提过API服务器对外暴露的一些不表示资源的URL路径(例如/healthz).常规**Role**和**RoleBinding**不能对这些类型的URL进行授权.但是**ClusterRole**可以.

##### 创建ClusterRole

命令格式和Role类似:`kubectl create clusterrole 名字`.下面这个例子创建一个pv-reader的集群角色,允许只读访问persistentvolumes资源

```bash
[root@k8s-master ~]# kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes
clusterrole.rbac.authorization.k8s.io/pv-reader created

[root@k8s-master ~]# kubectl get clusterrole pv-reader -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: "2020-05-05T10:26:01Z"
  name: pv-reader
  resourceVersion: "10725342"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/pv-reader
  uid: 2c6d3083-3385-46b3-9ea6-b479f57e7db5
rules:
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
[root@k8s-master ~]#
```

##### 创建ClusterRoleBinding

命令和RoleBinding使用方法类似: `kubectl create clusterrolebind`

下面的命令创建了个一个pv-test的`ClusterRoleBinding`集群角色绑定对线,指定的角色是`pv-reader`,分配给的ServiceAccount是`foo`名称空间下的default默认ServiceAccount

```
[root@k8s-master ~]# kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default
clusterrolebinding.rbac.authorization.k8s.io/pv-test created
```

现在foo名称空间下的Pod就可以访问集群下的persistentvolume(PV)资源了

```bash
/ # curl localhost:8001/api/v1/persistentvolumes
{
  "kind": "PersistentVolumeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/persistentvolumes",
    "resourceVersion": "10858720"
  },
  "items": [
    {
      "metadata": {
        "name": "pv-1",
        "selfLink": "/api/v1/persistentvolumes/pv-1",
        "uid": "544a9c07-1c41-41c5-8ed4-da41b4d11cf5",
        "resourceVersion": "10245103",
        "creationTimestamp": "2020-05-02T02:54:43Z",
        .......
```

> 由于指定了绑定到foo名称空间下的ServiceAccount.所以显而易见,bar名称空间下的pod没有权限访问perssitentvolumes资源

---

### ClusterRole授权访问指定命名空间中的资源

**ClusterRole**不是必须一直和集群级别的**ClusterRoleBinding**绑定使用.他们也可以和常规的**RoleBinding**进行捆绑.

在kubernetes中,有很多系统预定义的RBAC角色资源.通过`kubectl get`命令就可以看到系统预定义的角色.

```
kubectl get role -n kube-system
kubectl get clusterrole
kubectl get rolebinding -n kube-system
kubectl get clusterrolebinding
```

下面是一个系统预定义的名为view的集群角色

```bash
[root@k8s-master ~]# kubectl get clusterrole view -o yaml

  name: view
  resourceVersion: "413"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/view
  uid: 0aa05d0e-69b3-45df-8f92-be922386dbfe
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - persistentvolumeclaims/status
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - services/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  .......
```

​		这个ClusterRole又很多规则,允许只读(get,list,watch)访问各种资源.这些资源都是有名称空间的.这个名为View的ClusterRole的作用取决于它是和**ClusterRoleBinding**绑定还是和**RoleBinding**绑定.

* 如果创建了一个**ClusterRoleBinding**,并且引用了View这个**ClusterRole**.那么绑定的主体可以在所有名称空间中查看指定资源

* 如果创建的是一个**RoleBinding**,并且引用了View这个**ClusterRole**,那么在绑定中列出的主体只能查看在RoleBinding名称空间中的资源.

  

下面观察一下一个ClusterRole绑定到**ClusterRoleBinding**和**RoleBinding**的不同效果

在实验之前,确认一下默认情况下foo名称空间下的pod无论是访问集群下的pod还是名称空间下的pod都没有权限

```bash
/ # curl localhost:8001/api/v1/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
  
  
/ # curl localhost:8001/api/v1/namespaces/foo/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"foo\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
```



#### 1.ClusterRole绑定到ClusterRoleBinding

下面的命令将view这个ClusterRole绑定到foo名称空间下的ServiceAccount

```
[root@k8s-master ~]# kubectl create clusterrolebinding view-test --clusterrole=view --serviceaccount=foo:default
clusterrolebinding.rbac.authorization.k8s.io/view-test created
```

现在pod能列出foo名称空间下的Pod了

```
/ # curl localhost:8001/api/v1/namespaces/foo/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/foo/pods",
    "resourceVersion": "10865044"
  },
  "items": [
    {
      "metadata": {
        "name": "test-7b4bb6b9ff-qpppf",
```

而且也可以列出bar名称空间下的Pod了

```
/ # curl localhost:8001/api/v1/namespaces/bar/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/bar/pods",
    "resourceVersion": "10865200"
  },
  "items": [
    {
      "metadata": {
        "name": "test-7b4bb6b9ff-jl6v5"
```

还可以使用/api/v1/pods的URL路径来检索所有名称空间中的pod

```
/ # curl localhost:8001/api/v1/pods | more
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods",
    "resourceVersion": "10872966"
  },
  "items": [
    {
      "metadata": {
        "name": "test-7b4bb6b9ff-jl6v5",
        "generateName": "test-7b4bb6b9ff-",
        "namespace": "bar",
        ......
```

**总结:** 正如预期的那样.将ClusterRole绑定到ClusterRoleBinding.便可以获取整个集群中(包括所有名称空间)下的所有pod的列表.

---

#### 2.将ClusterRole绑定到RoleBinding.

先删除刚才创建的view-test的绑定关系删除

```
[root@k8s-master ~]# kubectl delete clusterrolebinding view-test
clusterrolebinding.rbac.authorization.k8s.io "view-test" deleted
```

创建一个RoleBinding替代

```
[root@k8s-master ~]# kubectl create rolebinding view-test --clusterrole=view --serviceaccount=foo:default -n foo
rolebinding.rbac.authorization.k8s.io/view-test created
```

现在foo名称空间有一个RoleBinding,将同一个名称空间中的default ServiceAccount绑定到view这个**ClusterRole**

现在pod仍然可以访问当前名称空间下的Pod

```
/ # curl localhost:8001/api/v1/namespaces/foo/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/foo/pods",
    "resourceVersion": "10876697"
  },
  "items": [
    {
      "metadata": {
        "name": "test-7b4bb6b9ff-qpppf",
```

但是现在已经无法访问bar名称空间下的pod了

```
/ # curl localhost:8001/api/v1/namespaces/bar/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"bar\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
```

整个集群下的pod资源当然也无法访问

```
/ # curl localhost:8001/api/v1/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
```

---

### 总结Role,ClusterRole,RoleBinding和ClusterRoleBinding的组合

| 访问资源                                                     | 角色类型    | 绑定类型           |
| ------------------------------------------------------------ | ----------- | ------------------ |
| 集群级别的资源(Nodes,PersistentVolumes,......)               | ClusterRole | ClusterRoleBinding |
| 非资源URL(/api,/healthz......)                               | ClusterRole | ClusterRoleBinding |
| 在任何名称空间中的资源(跨所有名称空间的资源)                 | ClusterRole | ClusterRoleBinding |
| 具体名称空间中的资源(在多个名称空间中重用这个相同的ClusterRole | ClusterRole | RoleBinding        |
| 在具体名称空间中的资源(Role必须在每个名称空间中定义)         | Role        | RoleBinding        |

---

### 了解默认的ClusterRole和ClusterRoleBinding

Kubernetes提供了默认的ClusterRole和ClusterRoleBinding.API服务器启动时都会更新他们,这保证了如果你错误地删除角色和绑定,Kubernetes会重新创建默认的角色和绑定

`kubectl get clusterrolebinding`和`kubectl get clusterrole`命令显示了系统默认的ClusterRole和ClusterRoleBinding

在前面的例子中,使用了默认的view ClusterRole.它允许读取一个名称空间中的大多数资源(除了Role,RoleBinding,Secret).

下面是一些其他的默认**ClusterRole**:

```
#允许修改一个名称空间中的资源,不允许查看和修改Role,RoleBinding
ClusterRole: edit

#一个名称空间中的资源完全控制权.可以读取和修改名称空间中的任何资源.和上面的区别在于是否有权限查看修改Role,RoleBinding
ClusterRole: admin

#集群完全控制权限
ClusterRole: cluster-admin
```

显然将admin或者cluster-admin授予pod应用程序或者用户是一个坏主意,和安全问题一样,最好是仅仅给每个人提供他们所需的权限(最小权限原则)

---

### 总结:

```
#创建ServiceAccount
kubectl create serviceaccount 服务账号名 

#将ServiceAccount分配给pod
在pod的manifest配置文件中指定ServiceAccount名:
serviceAccountName: 服务账号名

#RBAC4种角色资源
Role: 常规角色.定义了对某个资源具有某种访问权限
RoleBinding: 常规角色绑定.定义了该角色绑定到哪个主体
ClusterRole: 集群角色.同上
ClusterRoleBinding: 集群角色绑定.同上

区别在于:
常规角色和集群角色绑定作用于某个名称空间下.
集群角色和集群角色绑定作用于整个集群,不受限于任何名称空间

#了解集群默认集群角色和集群角色绑定
```

