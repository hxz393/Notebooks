# Service模板

### NodePort类型

使用NodePort类型服务来发布端口适合大多数Deployment捆绑使用.

发布之后:

- 外网通过 NodeIP:NodePort 比如192.168.2.114:30123访问.

- 内网其他Pod通过 服务名:Port 比如kubia-nodeport:80访问.

- 在容器Pod内部通过 localhost:targetPort 比如localhost:8080访问.

```sh
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
  Namespace: dev             #命名空间
spec:
  type: NodePort             #类型为NodePort
  ports:
  - name: http               #端口名
    port: 80                 #供集群中其它容器访问的端口
    targetPort: 8080         #容器原生使用端口
    nodePort: 30123          #互联网访问端口
  - name: https
    port: 443
    targetPort: 8081
    nodePort: 30123
  selector:
    app: kubia
```

