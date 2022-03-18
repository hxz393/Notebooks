# Pod模板

```yaml
apiVersion: v1        #必选,版本号,例如v1
kind: Pod       #必选,类型为Pod
metadata:       #必选,元数据
  name: string        #必选,Pod名称
  namespace: string     #必选,Pod所属的命名空间,默认为default
  labels:       #自定义标签
    - name: string      #自定义标签名字
  annotations:        #自定义注释列表
    - name: string    #自定义注释键值对
spec:         #必选,Pod中容器的详细定义
  containers:       #必选,Pod中容器列表
  - name: string      #必选,容器名称
    image: string     #必选,容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent]  #获取镜像的策略: Alawys表示每次都尝试重新下载镜像 | IfnotPresent表示优先使用本地镜像,本地不存在则下载镜像 | Nerver表示仅使用本地镜像
    command: [string]     #容器的启动命令列表,如不指定使用打包时使用的启动命令
    args: [string]       #容器的启动命令参数列表
    workingDir: string      #容器的工作目录
    volumeMounts:     #挂载到容器内部的存储卷配置
    - name: string      #引用pod定义的共享存储卷的名称,需用volumes[]部分定义的的卷名
      mountPath: string     #存储卷在容器内mount的绝对路径,应少于512字符
      subPath: string       #储存卷中的文件夹路径
      readOnly: boolean     #是否为只读模式,默认值为读写模式
    ports:        #需要暴露的端口号列表
    - name: string      #端口号名称
      containerPort: int    #容器需要监听的端口号
      hostPort: int         #容器所在主机需要监听的端口号,默认与Container相同.设置hostPort时,同一台node无法启动第二个副本
      protocol: string      #端口协议,支持TCP和UDP.默认TCP
    env:        #容器运行前需设置的环境变量列表
    - name: string      #环境变量名称
      value: string     #环境变量的值
    resources:        #资源限制和请求的设置
      limits:         #资源限制的设置
        cpu: string         #Cpu的限制,单位为core数,将用于docker run --cpu-shares参数
        memory: string      #内存限制,单位可以为Mib/Gib,将用于docker run --memory参数
      requests:       #资源请求的设置
        cpu: string         #Cpu请求,容器启动的初始可用数量
        memory: string      #内存请求,容器启动的初始可用数量
    livenessProbe:      #对Pod内的容器健康检查的设置,当探测无响应几次后将自动重启该容器,检查方法有exec,httpGet和tcpSocket,对一个容器只需设置其中一种方法
      exec:       #对Pod容器内检查方式设置为exec方式
        command: [string]   #exec方式需要制定的命令或脚本
      httpGet:        #对Pod内的容器健康检查方法设置为HttpGet,需要指定Path,port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:      #对Pod内的容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0   #容器启动完成后首次探测的时间,单位为秒
       timeoutSeconds: 0    #对容器健康检查探测等待响应的超时时间,单位秒,默认1秒.超过时间将重启容器
       periodSeconds: 0     #对容器监控检查的定期探测时间设置,单位秒,默认10秒1次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
    restartPolicy: [Always | Never | OnFailure] #Pod的重启策略,Always表示始终重启,OnFailure表示只有Pod以非0退出码退出才重启,Nerver表示不再重启该Pod
    nodeSelector: obeject   #设置NodeSelector表示将该Pod调度到包含这个标签的工作节点上,以key: value的格式指定
    imagePullSecrets:     #Pull镜像时使用的secret名称,以key: secretkey格式指定
    - name: string
    hostNetwork: false      #是否使用主机网络模式,默认为false,如果设为true,表示使用宿主机网络,不使用Docker网桥,该pod只能在同一节点启动一份
    volumes:        #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称,containers[].volumeMounts[].name将引用此名称
      emptyDir: {}      #类型为emtyDir的存储卷,与Pod同生命周期的一个临时目录.为空对象
      hostPath: string      #类型为hostPath的存储卷,表示挂载Pod所在宿主机的目录
        path: string      #Pod所在宿主机的目录,将被用于同期中mount的目录
      secret:       #类型为secret的存储卷,挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:      #类型为configMap的存储卷,挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string    
```

