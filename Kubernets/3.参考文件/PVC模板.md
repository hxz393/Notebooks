# PVC模板

用来绑定PV可供给多个pod使用

```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  labels:
    type: "SSD"               #PVC也能打标签
  namespace: default          #PVC有命名空间,只有同命名空间Pod才可用
spec:
  accessModes:                #访问模式,选项同PV
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi            #请求储存容量
  storageClassName: ""        
  selector:                   #选择PV标签
    matchLabels:
      type: ssd
```

