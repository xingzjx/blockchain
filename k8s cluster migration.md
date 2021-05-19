<h1>k8s从云端迁移到本地</h1>

- [kubernetes](#kubernetes)
  - [创建](#创建)
  - [连接](#连接)
  - [Dashboard](#dashboard)
    - [获取 Token](#获取-token)
    - [查看 dashboard](#查看-dashboard)
  - [基础设施](#基础设施)
  - [验证部分服务](#验证部分服务)
- [数据迁移](#数据迁移)
  - [测试数据库连接](#测试数据库连接)
  - [mysqldump](#mysqldump)
  - [上传数据到mysql服务器](#上传数据到mysql服务器)
  - [导入mysql](#导入mysql)
- [修改kubeconfig](#修改kubeconfig)
- [部署所有服务](#部署所有服务)
- [修改域名](#修改域名)

# kubernetes

> 本地测试集群

## 创建

使用kubeadm创建本地集群， [参考链接](https://makeoptim.com/service-mesh/kubeadm-kubernetes-istio-setup)

## 连接

下载 kubectl config 凭据

安装kubecm工具

```zsh
❯ brew install sunny0826/tap/kubecm
```

使用kubecm添加config凭据

```zsh
❯ kubecm add -f mb-local.conf -c
```

## Dashboard

### 获取 Token

```zsh
❯ kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-qp68s
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: c30d00af-d5d0-446c-94d3-6532ca7dd5b1

Type:  kubernetes.io/service-account-token

Data
====
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InNidktPWWNjYXM0ZTNFSmVzcXlldTZYQXMzQnBidDFrZ01zMUdWSWp5anMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTljdzl6Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI4MzljN2Y0Yy0zMmVhLTQ0MTgtYWNjYS0xNWNjNWIwNDliYzIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.mkPQefaBAwyaf187NvANXKYDbeeUv6Fs_OJv9qnwlqelHz6-pJ8Wl9m8xzJPX5wOSTnCxkzP4RCh5EYiYYkTL0sbWRY3XjPIVATV1cYHF0jM7aMIIaOL0Pk1qQOk4ftbm6nBJYIdCH4hhJf-tTaa9OyqmIO-tMxI67rFbEOdM1SCOP6uouve7Myq8kMOSygSWpAGJMLi6PDMfKeriy0GEG78whR2eqEL6FEhmpXWsC41IE_mDiFWKAMd9xld2IUEMjrdPlAuoUsbh5dRogeHQg13X43s33IiT8kxuR8JVjL6H_ROabGJL8_5Xdrb09HZcPyXIKx-LlYPQ3aNgposLA
```

### 查看 dashboard

```zsh
❯ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

打开 <http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/>，填入 `token` 即可查看 Dashboard。

## 基础设施

准备数据库，包括mysql ， mongodb和redis，在configuration定义好host，password等。

部署顺序

- [namespace](./namespace/README.md): 命名空间
- [tls](./tls/README.md): HTTPS（TLS）证书
- [istio](../sm-mb-we-test/istio/README.md): 部署 Istio
- [configuration](./configuration/README.md): 配置文件


注意：istio降级，需要重新部署tls以及执行operation下的restart-test.sh，重启service

## 验证部分服务

使用SwitchHosts工具添加域名和ip，然后进入工程，可以使用命令部署服务


# 数据迁移

已准备一台数据库服务器，已经安装mysql，mongodb和redis，本地安装好mysql，mongodb客户端以及redis，然后

## 测试数据库连接

mysql -uroot -pyourpassword -hyourip

mongo yourip:27017

redis-cli -h  yourip

## mysqldump

首先，查询出所有数据表，然后排除掉不需要迁移的数据库，执行如下命令，把数据dump到本地，

```zsh
❯ mysqldump -h'yourip' -uroot -p'yourpassword' --databases --column-statistics=0 --skip-lock-tables test1 test2>alldb.sql
```

## 上传数据到mysql服务器

注意不要在本地直接导入数据库到mysql服务器，这样会很慢。可以使用ftp工具或者shell工具上传到myql服务器，比如finalshell。


## 导入mysql

通过shell工具，登入mysql客户端：

```
> mysql -uroot -pyourpassord -hyourip

```

导入数据：


```
> source alldb.sql

```

# 修改kubeconfig

# 部署所有服务

登入jinkins，进入服务的流水线，找到deploy步骤中的重启deploy按钮，然后发布服务。把所有需要迁移的服务都执行一遍。

注意：迁移过程遇到资源不足问题

- 移除无用的资源和服务
- 限制pod数量
- 限制cpu和内存使用

# 修改域名

修改域名后，使用curl工具验证服务是否正常运行。


