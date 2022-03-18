# PV模板

## NFS

PV是全局资源,可以被任意PVC绑定.一般只用来绑定有状态应用,比如存放Gitlab,Jenkins数据.事先在卷内创建目录,再绑定到Pod.

```sh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
  labels:                                  #可以给PV打标签
    type: ssd
spec:
  capacity:                                
    storage: 10Gi                          #储存容量
  persistentVolumeReclaimPolicy: Retain    #卷回收策略,默认c,还有Recycle,Delete自动清空和删除
  accessModes:
  - ReadOnlyMany                           #允许多客户读取
  - ReadWriteOnce                          #允许单客户读写
  - ReadWriteMany                          #允许多客户读写
  nfs:                                     #储存类型是NFS
    path: /data                            #NFS储存已挂载的目录
    server: 192.168.2.113                  #NFS服务器IP
```

