# Node Affinity

Pod を特定の Node に配置するために Node Affinity を設定する。

今回は CPU bound な処理をする Pod を c5 系のノードに割り当てる。前提として eksctl で必要な nodegroup を作成しておく。

```
$ eksctl get nodegroup --cluster hello-world
CLUSTER        NODEGROUP      CREATED                 MIN SIZE    MAX SIZE    DESIRED CAPACITY    INSTANCE TYPE    IMAGE ID
hello-world    nodegroup-1    2020-06-20T13:28:26Z    1           3           1                   m5.large         ami-03964931fd94c2743
hello-world    nodegroup-2    2020-06-21T04:16:49Z    1           3           1                   c5.large         ami-03964931fd94c2743
```

## ラベルの確認

まず Node のラベルを確認する。今回欲しいインスタンスタイプの情報は `beta.kubernetes.io/instance-type` に入っている。

```
$ k get no
NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-59-152.ap-northeast-1.compute.internal   Ready    <none>   20m   v1.15.11-eks-af3caf
ip-192-168-60-145.ap-northeast-1.compute.internal   Ready    <none>   15h   v1.15.11-eks-af3caf

$ k get no/ip-192-168-60-145.ap-northeast-1.compute.internal -o json | jq '.metadata.labels'
{
  "alpha.eksctl.io/cluster-name": "hello-world",
  "alpha.eksctl.io/instance-id": "i-0a177de60093fb89f",
  "alpha.eksctl.io/nodegroup-name": "nodegroup-1",
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/instance-type": "m5.large",
  "beta.kubernetes.io/os": "linux",
  "failure-domain.beta.kubernetes.io/region": "ap-northeast-1",
  "failure-domain.beta.kubernetes.io/zone": "ap-northeast-1d",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "ip-192-168-60-145.ap-northeast-1.compute.internal",
  "kubernetes.io/os": "linux"
}

$ k get no/ip-192-168-59-152.ap-northeast-1.compute.internal -o json | jq '.metadata.labels'
{
  "alpha.eksctl.io/cluster-name": "hello-world",
  "alpha.eksctl.io/instance-id": "i-053cd19eee0021bc2",
  "alpha.eksctl.io/nodegroup-name": "nodegroup-2",
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/instance-type": "c5.large",
  "beta.kubernetes.io/os": "linux",
  "failure-domain.beta.kubernetes.io/region": "ap-northeast-1",
  "failure-domain.beta.kubernetes.io/zone": "ap-northeast-1d",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "ip-192-168-59-152.ap-northeast-1.compute.internal",
  "kubernetes.io/os": "linux"
}
```

ビルトインで付与されるラベルは次のとおり。 OS や CPU の命令セットに関するラベルもある。

* `kubernetes.io/hostname`
* `failure-domain.beta.kubernetes.io/zone`
* `failure-domain.beta.kubernetes.io/region`
* `beta.kubernetes.io/instance-type`
* `kubernetes.io/os`
* `kubernetes.io/arch`

## Node Affinity を設定する

今回は c5.large の方で実行するように Node Affinity を設定する。ちなみに、設定する前は 2 つのノードに均等に配置されているのがわかる。

```
$ k get po -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
nginx-deployment-68c7f5464c-f7r6c    ip-192-168-59-152.ap-northeast-1.compute.internal
nginx-deployment-68c7f5464c-f9d46    ip-192-168-59-152.ap-northeast-1.compute.internal
nginx-deployment-68c7f5464c-lhfh8    ip-192-168-60-145.ap-northeast-1.compute.internal
nginx-deployment-68c7f5464c-pwx7p    ip-192-168-60-145.ap-northeast-1.compute.internal
```

PodSpec に `nodeAffinity` の設定を追加する。 `requiredDuringSchedulingIgnoredDuringExecution` を指定すると条件に一致するノードにしか配置されない。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/instance-type
                operator: In
                values:
                - c5.large
```

apply すると条件に一致しなくなった 2 つの Pod がすぐさま作り直され、すべての Pod が c5.large のノードに再配置された。

```
$ k get po
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-68c7f5464c-f9d46   1/1     Running             0          26m
nginx-deployment-68c7f5464c-lhfh8   1/1     Running             0          26m
nginx-deployment-68c7f5464c-pwx7p   1/1     Running             0          26m
nginx-deployment-6c5c8b94f8-b29lc   0/1     ContainerCreating   0          4s
nginx-deployment-6c5c8b94f8-vghw6   0/1     ContainerCreating   0          4s

$ k get po -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
nginx-deployment-6c5c8b94f8-b29lc    ip-192-168-59-152.ap-northeast-1.compute.internal
nginx-deployment-6c5c8b94f8-n9lhr    ip-192-168-59-152.ap-northeast-1.compute.internal
nginx-deployment-6c5c8b94f8-rvb6l    ip-192-168-59-152.ap-northeast-1.compute.internal
nginx-deployment-6c5c8b94f8-vghw6    ip-192-168-59-152.ap-northeast-1.compute.internal
```

条件に一致するノードがない場合にどうなるかも検証する。 c5.large のノードを Cordon して Pod がスケジューリングされないようにする。

```
$ k cordon ip-192-168-59-152.ap-northeast-1.compute.internal
node/ip-192-168-59-152.ap-northeast-1.compute.internal cordoned

$ k get no
NAME                                                STATUS                     ROLES    AGE     VERSION
ip-192-168-59-152.ap-northeast-1.compute.internal   Ready,SchedulingDisabled   <none>   4h16m   v1.15.11-eks-af3caf
ip-192-168-60-145.ap-northeast-1.compute.internal   Ready                      <none>   19h     v1.15.11-eks-af3caf
```

この状態で再度 apply すると、すべての Pod が Pending になる。

```
$ k delete deploy/nginx-deployment
deployment.extensions "nginx-deployment" deleted

$ k apply -f nginx.yaml
deployment.apps/nginx-deployment created

$ k get po
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6c5c8b94f8-8fsq9   0/1     Pending   0          28s
nginx-deployment-6c5c8b94f8-b92x7   0/1     Pending   0          28s
nginx-deployment-6c5c8b94f8-hbv8c   0/1     Pending   0          28s
nginx-deployment-6c5c8b94f8-sbsgn   0/1     Pending   0          28s
```

Pod のイベントを見ると次のような Warning が出ており、空いている m5.large のノードに割り当てられなかったことがわかる。

> 0/2 nodes are available: 1 node(s) didn't match node selector, 1 node(s) were unschedulable.

このクラスタは Cluster Autoscaler を実行していたので、数分後に c5.large のノードが増えて Running になった。

## 優先度を付けた Node Affinity を設定する

実際の運用では、c5.large に割り当てられなかったら m5.large に割り当てたほうが可用性は上がる。このように優先順位を付けるには `preferredDuringSchedulingIgnoredDuringExecution` を使う。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: beta.kubernetes.io/instance-type
                operator: In
                values:
                - c5.large
          - weight: 2
            preference:
              matchExpressions:
              - key: beta.kubernetes.io/instance-type
                operator: In
                values:
                - m5.large
```

先ほどと同じように c5.large のノードを Cordon して apply しても、今度はすぐに Running になる。割り当てられたノードを見ると m5.large に配置されたことがわかる。

```
$ k get po -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
nginx-deployment-7ffc895f84-7bzdz    ip-192-168-60-145.ap-northeast-1.compute.internal
nginx-deployment-7ffc895f84-c277w    ip-192-168-60-145.ap-northeast-1.compute.internal
nginx-deployment-7ffc895f84-gb28r    ip-192-168-60-145.ap-northeast-1.compute.internal
nginx-deployment-7ffc895f84-kx7bz    ip-192-168-60-145.ap-northeast-1.compute.internal
```
