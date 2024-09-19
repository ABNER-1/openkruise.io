---
slug: openkruise-1.8
title: OpenKruise V1.8 版本解读：新增Advanced Statueful Set支持pvc自动变配能力
authors: [ Abner ]
tags: [ release ]
---

# advanced statefulset 支持pvc自动变配能力

Advanced Statefulset 新增 VolumeClaimUpdateStrategy 字段来实现 pvc 自动变配，定义如下所示：

```yaml
spec:
  volumeClaimUpdateStrategy:
    # 可选值有 OnPodRollingUpdate 和 OnDelete
    #   OnPodRollingUpdate ：controller 会在 pod 升级过程中自动扩容 pvc
    #   OnDelete ：controller 只会在 pvc 删除后使用新的 volume claim template 重建 pvc
    #             即 kruise 1.7 版本之前默认的 pvc 更新策略。
    type: OnPodRollingUpdate 
```

## 限制

pvc 只有在对应的 CSI 支持自动变配的情况下才可以实现扩容，且需要管理员配置对应的 storage class
`allowVolumeExpansion: true`。
这意味着：

1. 如果 pvc 对应的不是 CSI 插件，则无法实现自动变配；
2. 如果 pvc 对应的 CSI 没有实现 resize 的接口，则无法实现自动变配。
3. 如果 storage class 的 `allowVolumeExpansion` 字段为 false，则无法实现自动变配。

kruise webhook 目前会在显式配置`spec.volumeClaimUpdateStrategy.type = OnPodRollingUpdate`时尽可能拦截对不支持扩容的 pvc
的修改请求，包括：

1. 存在任何不能 patch 的 volume claim template 修改；
2. 只变更 volume claim template 容量大小的场景也会在以下场景下被拒绝：
    1. pvc 对应的 storage class 的 `allowVolumeExpansion` 字段为 false；
    2. pvc 对应的 storage class 为空，且集群内存在的默认 storage class 的 `allowVolumeExpansion` 字段为 false；
    3. pvc 对应的 storage class 为空，且集群内存在多个默认 storage class ，最新的默认 storage class
       的 `allowVolumeExpansion` 字段为 false；
    4. pvc 对应的 storage class 为空，且集群内不存在的默认 storage class

但这不代表可以拦截掉所有情况，包括但不限于：

1. pvc 对应的 storage class 的 `allowVolumeExpansion` 字段被管理员误设置，和真实的 CSI 能力不一致；
2. `OnDelete` 模式下修复非容量大小字段后，再次修改为 `OnPodRollingUpdate` 开启自动扩容；
3. 更新到一半因为节点/集群配额不足导致的 pvc 扩容失败。

对于这类错误场景，kruise 会卡住 pvc / pod 的升级进度以阻止进一步的错误扩散，需要用户介入后分析进行修复。

1. 对于不支持容量变配的 pvc 理论上永远不会变配成功，可以直接通过回滚 volume claim template 配置即可恢复。
2. 对于因各种配额问题导致的扩容失败，可以通过回滚 volume claim template 容量配置恢复
    - 此种情况下，已经扩容的 pvc 不会再被缩容
    - 后续在 k8s 集群中开启 1790 特性，可以再次被缩容
3. 历史各种原因导致 advanced statefulset 托管的 pvc 配置不一致情况下（特指存在不能通过原地patch的字段差异），因部分 pvc
   没法扩容导致的 pvc/pod 更新阻断，可以有以下几种处理方式：
    - 确保该 pvc 数据无需备份，可直接delete pvc，controller 会根据最新配置重建 pvc
    - 手动对 pvc 数据进行备份完成后可直接delete pvc，controller 会根据最新配置重建 pvc
    - 在评估可以暂不处理pvc的情况下，降级到 `OnDelete` 模式，此时不会再因为 pvc 的更新失败而强阻塞

## 单pvc变配

首先构造一个使用 volume claim template 的最简单的 advanced statefulset.

```bash
cat <<EOF | kubectl create -f -
apiVersion: apps.kruise.io/v1beta1
kind: StatefulSet
metadata:
  name: test-resize-pvc
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Delete
    whenScaled: Retain
  podManagementPolicy: OrderedReady
  replicas: 3
  selector:
    matchLabels:
      baz: blah
  serviceName: test
  template:
    metadata:
      labels:
        baz: blah
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/abner1/nginx:1.14-4
        name: nginx
        volumeMounts:
        - mountPath: /data0
          name: data0
      readinessGates:
        - conditionType: InPlaceUpdateReady
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
      minReadySeconds: 0
      partition: 0
      podUpdatePolicy: InPlaceIfPossible
    type: RollingUpdate
  volumeClaimTemplates:
  - metadata:
      name: data0
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
      storageClassName: allow-volume-expansion
EOF
```

修改 volume claim template 的容量大小为 3Gi

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"3Gi"},{"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"}]'
```

此时可以在 advanced statefulset status 中查看到 pvc 状态信息正在逐步进行变更

```yaml
status:
  #...
  volumeClaims:
    - compatibleReadyReplicas: 0
      compatibleReplicas: 1
      volumeClaimName: data0
```

你也可以在 advanced statefulset 上查看对应的 pvc 修改事件信息，如

```bash
Normal  SuccessfulResize  6m15s              statefulset-controller  resize Claim data0-test-resize-pvc-2 Pod test-resize-pvc-2 in StatefulSet test-resize-pvc success
Normal  SuccessfulResize  5m8s               statefulset-controller  resize Claim data0-test-resize-pvc-1 Pod test-resize-pvc-1 in StatefulSet test-resize-pvc success
Normal  SuccessfulResize  3m52s              statefulset-controller  resize Claim data0-test-resize-pvc-0 Pod test-resize-pvc-0 in StatefulSet test-resize-pvc success
```

## 复用 pod 更新的策略

### 利用 partition 限制 pvc 扩容进度

修改 volume claim template 的容量大小为 4Gi,且设置 partition 为 1

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"4Gi"},{"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"},{"op": "replace", "path": "/spec/updateStrategy/rollingUpdate/partition", "value":1}]'
```

此时等待前两个 pvc 变配完成后，data0-test-resize-pvc-0 会因为 partition 设置保持在旧版本的容量。

将 partition 设置为 0，则 data0-test-resize-pvc-0 会扩容到目标容量

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/updateStrategy/rollingUpdate/partition", "value":0}]'
```

### 利用 paused 限制 pvc 扩容进度

将 volume claim template 扩大到 5Gi

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"5Gi"},{"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"}]'
```

并立刻执行 paused，限制变更继续进行

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/updateStrategy/rollingUpdate/paused", "value":true}]'
```

会看到已修改的 pvc 会继续变更，但不会继续推进其他 pvc 的变更。

```yaml
status:
  #...
  volumeClaims:
    - compatibleReadyReplicas: 1
      compatibleReplicas: 1
      volumeClaimName: data0
```

且会在去除 paused 设置后继续推进

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/updateStrategy/rollingUpdate/paused", "value":false}]'
```

## 和镜像/配置变更协同进行 pvc 变配

通过设置 `spec.volumeClaimUpdateStrategy.type = OnPodRollingUpdate` 自动在 pod 进行变更的过程中进行 pvc 变配。
一般过程是先更新 pod 的所有待更新 pvc，等待 pvc 变配完成或者进入 `FileSystemResizePending` 状态，则开始对应 pod 的变配过程。

原地升级镜像的同时进行 pvc 的扩容：

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", 
  "value":"registry.cn-hangzhou.aliyuncs.com/abner1/nginx:1.15-4"}, 
  {"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"6Gi"}, 
  {"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"}]'
```

修改环境变量进行重建升级的同时进行 pvc 的扩容：

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/env", "value":[{"name": "test", "value": "true"}]}, 
  {"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"7Gi"}, 
  {"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"}]'
```

查看 advanced statefulset 的相关事件信息可以发现 controller 的操作流程是先 resize pvc，再 delete pod，之后再 create pod。

```bash
  Normal  SuccessfulResize            47s (x4 over 87m)  statefulset-controller  resize Claim data0-test-resize-pvc-2 Pod test-resize-pvc-2 in StatefulSet test-resize-pvc success
  Normal  SuccessfulCreate            46s (x3 over 93m)  statefulset-controller  create Pod test-resize-pvc-2 in StatefulSet test-resize-pvc successful
  Normal  SuccessfulDelete            46s (x4 over 10m)  statefulset-controller  delete Pod test-resize-pvc-2 in StatefulSet test-resize-pvc successful
  Normal  SuccessfulResize            37s (x4 over 86m)  statefulset-controller  resize Claim data0-test-resize-pvc-1 Pod test-resize-pvc-1 in StatefulSet test-resize-pvc success
  Normal  SuccessfulDelete            37s (x2 over 10m)  statefulset-controller  delete Pod test-resize-pvc-1 in StatefulSet test-resize-pvc successful
  Normal  SuccessfulCreate            36s (x3 over 94m)  statefulset-controller  create Pod test-resize-pvc-1 in StatefulSet test-resize-pvc successful
  Normal  SuccessfulResize            27s (x4 over 59m)  statefulset-controller  resize Claim data0-test-resize-pvc-0 Pod test-resize-pvc-0 in StatefulSet test-resize-pvc success
  Normal  SuccessfulDelete            27s (x2 over 10m)  statefulset-controller  delete Pod test-resize-pvc-0 in StatefulSet test-resize-pvc successful
  Normal  SuccessfulCreate            26s (x3 over 94m)  statefulset-controller  create Pod test-resize-pvc-0 in StatefulSet test-resize-pvc successful
```

## 错误场景演示

### 可回滚的场景
因为配额错误导致的扩容失败场景不是很方便构造，就复用 partition 能力来构造扩容到中间无法继续扩容的错误场景。
```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/persistentVolumeClaimRetentionPolicy/whenDeleted", "value":"Delete"}]'
kubectl delete asts test-resize-pvc
cat <<EOF | kubectl create -f -
apiVersion: apps.kruise.io/v1beta1
kind: StatefulSet
metadata:
  name: test-resize-pvc
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Delete
    whenScaled: Retain
  podManagementPolicy: OrderedReady
  replicas: 3
  selector:
    matchLabels:
      baz: blah
  serviceName: test
  template:
    metadata:
      labels:
        baz: blah
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/abner1/nginx:1.14-4
        name: nginx
        resources:
          limits:
            cpu: 100m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - mountPath: /data0
          name: data0
      readinessGates:
        - conditionType: InPlaceUpdateReady
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
      minReadySeconds: 0
      partition: 0
      podUpdatePolicy: InPlaceIfPossible
    type: RollingUpdate
  volumeClaimTemplates:
  - metadata:
      name: data0
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
      storageClassName: allow-volume-expansion
EOF
```

更改镜像、扩容到 3Gi，设置 `spec.volumeClaimUpdateStrategy.type = OnPodRollingUpdate`，同时设置 partition 1.
```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", 
  "value":"registry.cn-hangzhou.aliyuncs.com/abner1/nginx:1.15-4"}, 
  {"op": "replace", "path": "/spec/updateStrategy/rollingUpdate/partition", "value":1},
  {"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"3Gi"}, 
  {"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"}]'
```

等待更新完成，此时 pvc0 仍然是2Gi， pvc1 和 pvc2则已经扩容到 3Gi，来模拟 pvc0 因为各种原因无法扩容而卡住。
此时 advanced statefulset 对 pvc 的跟踪状态可能如下所示：
```yaml
status:
  replicas: 3
  volumeClaims:
  - compatibleReadyReplicas: 2
    compatibleReplicas: 2
    volumeClaimName: data0
```

此时可以通过直接回滚容量配置来恢复。

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"2Gi"}, 
  {"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"}]'
```
等待片刻可以见到 advanced statefulset 的跟踪的 pvc 状态已经完全一致了。

```yaml
status:
  replicas: 3
  volumeClaims:
  - compatibleReadyReplicas: 3
    compatibleReplicas: 3
    volumeClaimName: data0
```
此时查看 pvc 状态, 可以看到 pvc-1 和 pvc-2的容量仍然为 3Gi。这主要是因为在 k8s 1.27 之前是不允许缩容 pvc 的。k8s 1.27 之后有一个门控 `RecoverVolumeExpansionFailure`目前在 alpha 阶段，开启之后可以支持缩容 pvc。advanced statefulset 会在未来计划支持该门控开启下的行为。

```bash
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
data0-test-resize-pvc-0   Bound    pvc-a66ce863-4e19-4fc4-b0a9-8c55fa1615e3   2Gi        RWO            allow-volume-expansion   4m3s
data0-test-resize-pvc-1   Bound    pvc-322fd753-026a-416e-b100-4659b4b11ca0   3Gi        RWO            allow-volume-expansion   3m54s
data0-test-resize-pvc-2   Bound    pvc-cede9cda-8fa1-4929-a693-4d705db8e4dd   3Gi        RWO            allow-volume-expansion   3m44s
```


### 不可回滚的场景

构造一个不可变配的 storage class, 如 disallow-volume-expansion `ALLOWVOLUMEEXPANSION` 为 false。

```shell
NAME                        PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
allow-volume-expansion      hostpath.csi.k8s.io     Delete          Immediate              true                   6d22h
disallow-volume-expansion   hostpath.csi.k8s.io     Delete          Immediate              false                  6d22h
```

构造一个残留的不可变配的 pvc

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/persistentVolumeClaimRetentionPolicy/whenDeleted", "value":"Delete"}]'
kubectl delete asts test-resize-pvc
kubectl delete pvc data0-test-resize-pvc-1
cat << EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    apps.kruise.io/owned-by-asts: test-resize-pvc
  labels:
    baz: blah
  name: data0-test-resize-pvc-1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: disallow-volume-expansion
EOF
```

创建一个使用这个残留 pvc 的 advanced statefulset

```bash
cat << EOF | kubectl create -f -
apiVersion: apps.kruise.io/v1beta1
kind: StatefulSet
metadata:
  name: test-resize-pvc
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Delete
    whenScaled: Retain
  podManagementPolicy: OrderedReady
  replicas: 3
  selector:
    matchLabels:
      baz: blah
  serviceName: test
  template:
    metadata:
      labels:
        baz: blah
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/abner1/nginx:1.14-4
        name: nginx
        volumeMounts:
        - mountPath: /data0
          name: data0
      readinessGates:
        - conditionType: InPlaceUpdateReady
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
      minReadySeconds: 0
      partition: 0
      podUpdatePolicy: InPlaceIfPossible
    type: RollingUpdate
  volumeClaimTemplates:
  - metadata:
      name: data0
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
      storageClassName: allow-volume-expansion
EOF
```

此时 advanced statefulset 的状态中显示的状态如下：

```yaml
status:
  readyReplicas: 3
  replicas: 3
  volumeClaims:
    - compatibleReadyReplicas: 2
      compatibleReplicas: 2
      volumeClaimName: data0
```

其中 compatibleReplicas 为2，小于 pod 的 replicas。原因是残留 pvc 和 volume claim template 并不匹配，但此时默认的
volumeClaimUpdateStrategy 为 OnDelete，并不阻断 pod 更新顺序，此时如果 delete 该残留 pvc 则会创建一个最新配置的 pvc。

接下来修改 volume claim template 的容量大小为 3Gi

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/volumeClaimTemplates/0/spec/resources/requests/storage", "value":"3Gi"},{"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnPodRollingUpdate"}]'
```

等待片刻，可以看到 advanced statefulset 的更新貌似并未进行，且 compatibleReplicas 为 0。
此时如果查看 kruise manager 的日志，就会发现是因为 test-resize-pvc-1 因为不匹配，pod1 被认为不可用pod，突破了最大不可用数，所以被阻断了。

```yaml
status:
  readyReplicas: 3
  replicas: 3
  volumeClaims:
    - compatibleReadyReplicas: 0
      compatibleReplicas: 0
      volumeClaimName: data0
```

此时可行的两种解决方案是：

1. 在评估数据安全后，确保数据备份之后删除对应的 pvc 和 pod，按最新的配置重建 ready 后继续更新
2. 关闭`OnPodRollingUpdate`，以避免阻断紧急的更新需求。

这里将演示第二种:

1. 首先更新一个镜像来模拟紧急更新需求。

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", 
  "value":"registry.cn-hangzhou.aliyuncs.com/abner1/nginx:1.15-4"}]'
```

此时表现依旧是

```yaml
status:
  availableReplicas: 3
  currentReplicas: 3
  observedGeneration: 3
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 0   # pod 没法更新
  volumeClaims:
    - compatibleReadyReplicas: 0
      compatibleReplicas: 0
      volumeClaimName: data0
```

2. 修改 volumeClaimUpdateStrategy 为 OnDelete

```bash
kubectl patch asts test-resize-pvc --type='json' -p='[{"op": "replace", "path": "/spec/volumeClaimUpdateStrategy/type", "value":"OnDelete"}]'
```

再次观测，变更如预期一样进行。变更完成后的 status 如下：

```yaml
status:
  availableReplicas: 3
  currentReplicas: 3
  observedGeneration: 4
  readyReplicas: 3
  replicas: 3
  updatedAvailableReplicas: 3 # pod 更新完成
  updatedReadyReplicas: 3
  updatedReplicas: 3
  volumeClaims:
    - compatibleReadyReplicas: 0 # 但 pvc 依旧未更新，但因为 OnDelete 策略不再阻断。
      compatibleReplicas: 0
      volumeClaimName: data0
```

## 不只是 resize pvc

resize pvc 自从 pvc api 支持 expand 之后就一直被社区提议增加到 statefulset 之中，但同时也有一种声音：
为什么不是运维操作，由一个上层组件来批量 patch pvc 实现扩容行为呢？
这方面的我们也进行了思考：

1. 存储资源的大小变配可以视为容量变更操作，我们认为支持灰度是一个更好的选择，可以避免一些边界错误的进一步扩散
2. 除此之外，我们注意到目前的工作负载对于一些存储的特性的集成不够，比如高版本 CSI 实现的 `volumeSnapshot`
   可以用来帮助用户更方便的备份存储资源，但是如何需要将快照结果还原到 workload
   当中，目前是一个值得探索的问题？现在基于云盘大小变配实现了`OnPodRollingUpdate`的更新模式，理论上是可以通过集成快照等能力，进行更多的无损存储变配操作。

# 社区参与

非常欢迎你通过 Github/Slack/钉钉/微信 等方式加入我们来参与 OpenKruise 开源社区。
你是否已经有一些希望与我们社区交流的内容呢？
可以在我们的[社区双周会](https://shimo.im/docs/gXqmeQOYBehZ4vqo)上分享你的声音，或通过以下渠道参与讨论：

- 加入社区 [Slack channel](https://kubernetes.slack.com/channels/openkruise) (English)
- 加入社区钉钉群：搜索群号 `23330762` (Chinese)
- 加入社区微信群（新）：添加用户 `openkruise` 并让机器人拉你入群 (Chinese)

