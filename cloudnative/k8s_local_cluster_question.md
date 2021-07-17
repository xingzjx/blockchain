# k8s自建集群问题总结


## caligo　网络问题

经过验证相同 node　下的　pod　可以相互通信，不同　node 的　pod　无法相互访问，定位到是　calico　异常

```bash

calicoctl node status

```

查看　pod 日志

```bash

failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket: dial unix /var/run/calico/bird.ctl: connect: connection refused

```

设置　IP_AUTODETECTION_METHOD　参数，编辑　[calico.yaml](https://docs.projectcalico.org/v3.12/manifests/calico.yaml)　找到　name　: IP，在它上面写入：

```yaml

    - name: CLUSTER_TYPE
        value: "k8s,bgp"
    # 新增部分
    - name: IP_AUTODETECTION_METHOD
        value: "interface=eno1"
    # value: "interface=eth.*"，　interface　是自己机器网卡的名字
    # value: "interface=can-reach=www.baidu.com"
    # 新增部分结束
    - name: IP
        value: "autodetect"
    - name: CALICO_IPV4POOL_IPIP
        value: "Always"

```

参考:

[calico 网络问题](https://my.oschina.net/u/1431757/blog/3144782)

[calico node bird 原理分析](https://blog.csdn.net/zhonglinzhang/article/details/97626768)

## Service 无法访问问题排查

参考：https://kubernetes.io/zh/docs/tasks/debug-application-cluster/debug-service/

## DNS 无法解析问题排查

参考：https://kubernetes.io/zh/docs/tasks/administer-cluster/dns-debugging-resolution/

## kubeadm init　问题

初始化集群命令：

```bash

sudo kubeadm init --apiserver-advertise-address=xx.xx.xx.xx --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all  --v=5


# ...省略...

 [kubelet-check] Initial timeout of 40s passed.

couldn not initialize a Kubernetes cluster
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/init.runWaitControlPlanePhase
	/workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/init/waitcontrolplane.go:114
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run.func1

# ...省略...

```

请检查 kubectl 以及　apiserver 是否正确。



