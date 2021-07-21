# k8s集群问题总结


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

参考：

[kubeadm reset 环境及网络清理](https://www.ziji.work/kubernetes/kubeadm-reset-network-errors.html)

## 删除 namespace Terminating 问题

首先把本机服务暴露在本地端口的8001端口上，执行如下命令

```

kubectl proxy

```

然后执行，delete_namespace.sh

```shell

my_namespace = 'eth-space'

kubectl get namespace eth-space -o json >tmp.json

sed 's/"kubernetes"//' tmp.json

curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/$my_namespace/finalize

```

参考：

[k8s的pod或者ns资源一直terminating删除办法](https://www.cnblogs.com/xiaoyaojinzhazhadehangcheng/p/12067283.html)

## metrics-server 

错误日志：no such host，编辑 metrics-server 的 yaml 文件，在 Deployment 添加　--kubelet-preferred-address-types=InternalIP

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: xingzjx/metrics-server:v0.5.0
        imagePullPolicy: IfNotPresent
        args: 
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
        - --cert-dir=/tmp
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
```



