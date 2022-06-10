Kubernetes 对于有状态的日期应用或者对于数据需要持久化的应用，就需要更可靠的存储来保存应用产生的重要数据了。这里需要的就是持久卷（PersistentVolume）和 持久卷申领（PersistentVolumeClaim）。

涉及的知识点：
- PV
- PVC
- StorageClass
- StatefulSet
- Service

持久卷（PersistentVolume，PV）是集群中的一块存储，可以事先供应，或者使用[存储类（Storage Class）](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)来动态供应。 持久卷是集群资源，就像节点也是集群资源一样。PV 持久卷和普通的 Volume 一样，也是使用 卷插件来实现的，只是它们拥有独立于任何使用 PV 的 Pod 的生命周期。 此 API 对象中记述了存储的实现细节，无论其背后是 NFS、iSCSI 还是特定于云平台的存储系统。

持久卷申领（PersistentVolumeClaim，PVC）表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）；同样 PVC 申领也可以请求特定的大小和访问模式 （例如，可以要求 PV 卷能够以 ReadWriteOnce、ReadOnlyMany 或 ReadWriteMany 模式之一来挂载，参见[访问模式](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)）。

下面以 influxdb 为例子：

```yaml
# 单独弄个Namespace influxdb
apiVersion: v1
kind: Namespace
metadata:
    name: influxdb
---
# 配置文件弄个ConfigMap，便于修改
apiVersion: v1
kind: ConfigMap
metadata:
  name: influxdb-config
  namespace: influxdb
data:
  influxdb.conf: >-
    [meta]
      dir = "/var/lib/influxdb/meta"

    [coordinator]
      log-queries-after = "60s"
      max-select-series = 10000
      write-timeout = "30s"
    [data]
      dir = "/var/lib/influxdb/data"
      engine = "tsm1"
      wal-dir = "/var/lib/influxdb/wal"
      cache-max-memory-size = "0"
      cache-snapshot-memory-size = "256m"
      index-version = "tsi1"
      max-index-log-file-size = "64k"
      max-series-per-database = 3000000
      max-values-per-tag = 100000

---
## PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: influxdb-data
  namespace: influxdb
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: csi-xxx # 需自定义
  resources:
    requests:
      storage: 16Gi # 按需配置，存储数据较多可适当放大
---
# influxdb 有状态应用
apiVersion: apps/v1
kind: StatefulSet
metadata:
    labels:
        app: influxdb
    name: influxdb
    namespace: influxdb
spec:
    replicas: 1
    selector:
        matchLabels:
            app: influxdb
    serviceName: influxdb
    template:
        metadata:
            labels:
                app: influxdb
        spec:
            containers:
              - image: influxdb:1.8.10
                name: influxdb
                ports:
                  - containerPort: 8086
                    name: influxdb
                volumeMounts:
                  - mountPath: /var/lib/influxdb
                    name: data
                  - name: influxdb-config
                    mountPath: /etc/influxdb/influxdb.conf
                    readOnly: true
                    subPath: influxdb.conf
            volumes:
              - name: data
                persistentVolumeClaim:
                  claimName: influxdb-data
              - name: timezone-config # 时区
                hostPath:
                  path: /usr/share/zoneinfo/Asia/Shanghai
              - name: influxdb-config
                configMap:
                  defaultMode: 0600
                  name: influxdb-config
---
# 提高端口，方便其他 pod 调用
apiVersion: v1
kind: Service
metadata:
    name: influxdb
    namespace: influxdb
spec:
    ports:
      - name: influxdb
        port: 8086
        targetPort: 8086
    selector:
        app: influxdb
    type: ClusterIP
```