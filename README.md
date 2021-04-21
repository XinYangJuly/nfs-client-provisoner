# nfs-client-provisoner

一、NFS Provisioner 简介

NFS Provisioner 是一个自动配置卷程序，它使用现有的和已配置的 NFS 服务器来支持通过持久卷声明动态配置 Kubernetes 持久卷。

持久卷被配置为：n a m e s p a c e − {namespace}-namespace−{pvcName}-${pvName}。

二、创建 NFS Server 端

本篇幅是具体介绍如何部署 NFS 动态卷分配应用 “NFS Provisioner”，所以部署前请确认已经存在 NFS Server 端，关于如何部署 NFS Server 请看之前写过的博文 “CentOS7 搭建 NFS 服务器”，如果非 Centos 系统，请先自行查找 NFS Server 安装方法。

这里 NFS Server 环境为：

IP地址：10.0.0.4
存储目录：/mnt/resource

三、创建 ServiceAccount

现在的 Kubernetes 集群大部分是基于 RBAC 的权限控制，所以创建一个一定权限的 ServiceAccount 与后面要创建的 “NFS Provisioner” 绑定，赋予一定的权限。

提前修改里面的 Namespace 值设置为要部署 “NFS Provisioner” 的 Namespace 名

nfs-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-client
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
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
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-client
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-client
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-client
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-client
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io


创建 RBAC
-n: 指定应用部署的 Namespace
 kubectl apply -f nfs-rbac.yaml

四、部署 NFS Provisioner

设置 NFS Provisioner 部署文件，这里将其部署到 “kube-system” Namespace 中。
nfs-provisioner-deploy.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-client
spec:
  replicas: 1
  strategy:
    type: Recreate    #---设置升级策略为删除再创建(默认为滚动更新)
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-client     #--- nfs-provisioner的名称，以后设置的storageclass要和这个保持一致
            - name: NFS_SERVER
              value: 10.0.0.4    #---NFS服务器地址，和 valumes 保持一致
            - name: NFS_PATH
              value: /mnt/resource    #---NFS服务器目录，和 valumes 保持一致
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.0.0.4    #---NFS服务器地址
            path: /mnt/resource    #---NFS服务器目录


创建 NFS Provisioner
-n: 指定应用部署的 Namespace
kubectl apply -f nfs-provisioner-deploy.yaml -n nfs-client


五、创建 NFS SotageClass

创建一个 StoageClass，声明 NFS 动态卷提供者名称为 “nfs-storage”。
nfs-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  namespace: nfs-client
provisioner: nfs-client     #---动态卷分配者名称，必须和上面创建的"provisioner"变量中设置的Name一致
parameters:
  archiveOnDelete: "true"   #---设置为"false"时删除PVC不会保留数据,"true"则保留数据


创建 StorageClass
kubectl apply -f nfs-storage.yaml


六、创建 PVC 和 Pod 进行测试

1、创建测试 PVC
在 “kube-public” Namespace 下创建一个测试用的 PVC 并观察是否自动创建是 PV 与其绑定。
test-pvc.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
  namespace: nfs-client
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi

创建 PVC
kubectl apply -f test-pvc.yaml


查看 PVC 状态是否与 PV 绑定
利用 Kubectl 命令获取 pvc 资源，查看 STATUS 状态是否为 “Bound”。
$ kubectl get pvc test-pvc -n kube-public

NAME       STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS
test-pvc   Bound    pvc-be0808c2-9957-11e9 1Mi        RWO            nfs-storage

2、创建测试 Pod 并绑定 PVC
创建一个测试用的 Pod，指定存储为上面创建的 PVC，然后创建一个文件在挂载的 PVC 目录中，然后进入 NFS 服务器下查看该文件是否存入其中。
test-pod.yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: gcr.io/google_containers/busybox:1.24
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim


创建 Pod
-n：指定创建 Pod 的 Namespace
$ kubectl apply -f test-pod.yaml -n kube-public


3、进入 NFS Server 服务器验证是否创建对应文件
进入 NFS Server 服务器的 NFS 挂载目录，查看是否存在 Pod 中创建的文件：

$ cd /mnt/resource
$ ls -l

-rw-r--r-- 1 root root 0 Apr 21 06:01 SUCCESS

