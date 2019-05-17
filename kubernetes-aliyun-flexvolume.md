---
title: Kubernetes aliyun flexvolume
layout: 2017/sheet
prism_languages: [bash,yaml]
weight: -3
tags: [Featured]
updated: 2017-11-19
category: Kubernetes
---

## Kubernetes aliyun flexvolume

{: .-one-column}

### 配置阿里云role

```json
{
    "Version": "1",
    "Statement": [
        {
            "Action": [
                "ecs:Describe*",
                "ecs:AttachDisk",
                "ecs:CreateDisk",
                "ecs:CreateSnapshot",
                "ecs:DeleteDisk",
                "ecs:DeleteSnapshot",
                "ecs:DetachDisk",
                "ecs:ModifyAutoSnapshotPolicyEx",
                "ecs:ModifyDiskAttribute"
            ],
            "Resource": [
                "*"
            ],
            "Effect": "Allow"
        },
        {
            "Action": [
                "nas:*"
            ],
            "Resource": [
                "*"
            ],
            "Effect": "Allow"
        },
        {
            "Action": [
                "oss:*"
            ],
            "Resource": [
                "*"
            ],
            "Effect": "Allow"
        }
    ]
}
```

{: .-one-column}

### 部署aliyun flexvolume

aliyun-flexvolume.yml 

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: flexvolume
  namespace: kube-system
  labels:
    k8s-volume: flexvolume
spec:
  selector:
    matchLabels:
      name: acs-flexvolume
  template:
    metadata:
      labels:
        name: acs-flexvolume
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: acs-flexvolume
        image: registry.cn-hangzhou.aliyuncs.com/acs/flexvolume:v1.9.7-42e8198
        imagePullPolicy: Always
        securityContext:
          privileged: true
        env:
        - name: ACS_DISK
          value: "true"
        - name: ACS_NAS
          value: "true"
        - name: ACS_OSS
          value: "true"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: usrdir
          mountPath: /host/usr/
        - name: etcdir
          mountPath: /host/etc/
        - name: logdir
          mountPath: /var/log/alicloud/
      volumes:
      - name: usrdir
        hostPath:
          path: /usr/
      - name: etcdir
        hostPath:
          path: /etc/
      - name: logdir
        hostPath:
          path: /var/log/alicloud/
---
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: alicloud-disk-common
provisioner: alicloud/disk
parameters:
  type: cloud
---
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: alicloud-disk-efficiency
provisioner: alicloud/disk
parameters:
  type: cloud_efficiency
---
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: alicloud-disk-ssd
provisioner: alicloud/disk
parameters:
  type: cloud_ssd
---
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: alicloud-disk-available
provisioner: alicloud/disk
parameters:
  type: available
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: alicloud-disk-controller-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alicloud-disk-controller
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: run-alicloud-disk-controller
subjects:
  - kind: ServiceAccount
    name: alicloud-disk-controller
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: alicloud-disk-controller-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: alicloud-disk-controller
  namespace: kube-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: alicloud-disk-controller
    spec:
      tolerations:
      - effect: NoSchedule
        operator: Exists
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        operator: Exists
        key: node.cloudprovider.kubernetes.io/uninitialized
      nodeSelector:
         node-role.kubernetes.io/master: ""
      serviceAccount: alicloud-disk-controller
      containers:
        - name: alicloud-disk-controller
          image: registry.cn-hangzhou.aliyuncs.com/acs/alicloud-disk-controller:v1.9.3-ed710ce
          volumeMounts:
            - name: cloud-config
              mountPath: /etc/kubernetes/
            - name: logdir
              mountPath: /var/log/alicloud/
      volumes:
        - name: cloud-config
          hostPath:
            path: /opt/kubernetes/conf/
        - name: logdir
          hostPath:
            path: /var/log/alicloud/
```

##  阿里云盘动态存储卷
**默认选项**

在多可用区的集群中，需要您手动创建上述 StorageClass，这样可以更准确的定义所需要云盘的可用区信息；
集群默认提供了下面几种 StorageClass，可以在单可用区类型的集群中使用。
- alicloud-disk-common：普通云盘。
- alicloud-disk-efficiency：高效云盘。
- alicloud-disk-ssd：SSD云盘。
- alicloud-disk-available：提供高可用选项，先试图创建高效云盘；如果相应可用区的高效云盘资源售尽，再试图创建SSD盘；如果SSD售尽，则试图创建普通云盘。

**规格限制**

对创建的云盘容量有如下要求：

- 普通云盘：最小5Gi
- 高效云盘：最小20Gi
- SSD云盘：最小20Gi




#### 创建 StorageClass 
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-ssd-beijing-a
parameters:
  regionid: cn-beijing
  type: cloud_ssd
  zoneid: cn-beijing-a
provisioner: alicloud/disk
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-ssd-beijing-b
parameters:
  regionid: cn-beijing
  type: cloud_ssd
  zoneid: cn-beijing-b
provisioner: alicloud/disk
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

参数说明：
- provisioner：配置为 alicloud/disk，标识StorageClass使用阿里云云盘 provisioner 插件创建。
- type：标识云盘类型，支持 cloud、cloud_efficiency、cloud_ssd、available 四种类型；其中 available 会对高效、SSD、普通云盘依次尝试创建，直到创建成功。
- regionid：期望创建云盘的区域。
- reclaimPolicy: 云盘的回收策略，默认为Delete，支持Retain。如果数据安全性要求高，推荐使用Retain方式以免误删；
- zoneid：期望创建云盘的可用区。
- encrypted：（可选）创建的云盘是否加密，默认情况是false，创建的云盘不加密。

#### 创建服务
```yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: disk-ssd
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: alicloud-disk-ssd-hangzhou-b
  resources:
    requests:
      storage: 20Gi
---
kind: Pod
apiVersion: v1
metadata:
  name: disk-pod-ssd
spec:
  containers:
  - name: disk-pod
    image: nginx
    volumeMounts:
      - name: disk-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: disk-pvc
      persistentVolumeClaim:
        claimName: disk-ssd
```



## 参考

- [容器服务支持 kubernetes Pod 自动绑定阿里云云盘、NAS、 OSS 存储服务。](https://help.aliyun.com/document_detail/86784.html?spm=a2c4g.11174283.6.682.24b12cee9AZwuM)
