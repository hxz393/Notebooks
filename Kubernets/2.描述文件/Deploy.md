# Deploy描述文件实例

```yaml
apiVersion: apps/v1                        #必选,早前版本有extensions/v1beta1或apps/v1beta1
kind: Deployment                           #必选,类型为Deployment
metadata:                                  #必选,元数据信息 
  name: myapp                              #必选,Deploy的名称
  namespace: pro                           #命名空间,实际部署到的环境
  labels:                                  #标签
    app: myapp                             #应用名
    version: v1                            #版本
    env: pro                               #发布环境
    tier: frontend                         #构架,区分前后端或base基础服务
    partition: HangZhou                    #服务器地区信息
    project: k8s                           #所属项目名
spec:                                      #必选,规格说明
  replicas: 3                              #必选,副本数量
  selector:                                #标签选择器,用来选择Pod
    matchLabels:                           #标签选择器
      app: myapp                           #选择标签键值对
    matchExpressions:                      #使用标签选择表达式,如果同时两种标签选择则必须都满足
      - {key: tier, operator: In, values: [cache]}  
  minReadySeconds: 10                      #启动多长时间内容器未发生崩溃等异常状态视为就绪,默认0秒
  template:                                #必选,定义Pod的模板
    metadata:                              
      labels:                              #生成Pod的标签
        app: myapp                         #可以和选择器一致
    spec:                                  
      terminationGracePeriodSeconds: 60    #发送SIGTERM信号后等待程序关闭的时间
      restartPolicy: Always                #重启策略
      serviceAccountName: flannel          #SA账号名
      nodeSelector:                       
        disktype: SSD    
      tolerations:                         #污点信息
      - effect: NoSchedule
        operator: Exists        
      containers:                          
      - name: myapp                        #Pod内运行容器名
        image: my.com/user/myapp:1.0       #拉取镜像地址
        imagePullPolicy: IfNotPresent      
        command:                           #容器启动后的命令,还可以用["sleep", "999999"]格式
        - sh
        - -C
        - 'while true; do echo "waiting"; sleep 5; done; echo "Done"'
        args:                              #命令参数
        - '-config.file=/etc/prometheus/prometheus.yml'
        ports:                             #容器端口
        - name: http                       #端口名称
          containerPort: 8080              #指定容器内端口
          hostPort: 811                    #指定映射到的节点端口,通过节点IP:端口即能访问
          protocol: TCP                    #定义协议,还可以是UDP
        volumeMounts:                      #挂载volumes中定义的磁盘
        - name: sdb                        #Volume的名称
          mountPath: /data/media           #挂载到容器中的路径
          subPath: /nginx/config           #挂载卷内目录的绝对路径
        env:                               #通过环境变量的方式,直接传递自定义环境变量
        - name: LOCAL_KEY                  #本地Key
          value: value                     #key值
        - name: CONFIG_MAP_KEY             #局策略可使用configMap的配置Key
          valueFrom:                       #指定来源
            configMapKeyRef:               #来自configMap
              name: special-config         #选择name为special-config的configmap
              key: special.type            #选择的key
        resources:                         #资源信息
          requests:                        #请求最小资源
            cpu: 0.1
            memory: 100Mi
          limits:                          #限制最大资源
            cpu: 0.2
            memory: 320Mi
        livenessProbe:                     #存活探针
          httpGet:                         #采用HTTP GET方式
            path: /health                  #监测路径
            port: 8080                     #端口
            scheme: HTTP                   #方式
          initialDelaySeconds: 60          #启动后延时多久开始运行监测
          timeoutSeconds: 5                #超时时间
          successThreshold: 1              #处于失败状态时,探测至少连续成功多少次转为成功
          failureThreshold: 5              #处于成功状态时,探测至少连续失败多少次转为失败
        readinessProbe:                    #就绪探针
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      volumes:                             #挂载卷信息
      - name: log-cache                  
        emptyDir: {}                       #挂载临时卷
      - name: sdb
        hostPath:                          #挂载节点上的目录
          path: /any/path                  #说明路径
      - name: example-volume-config        #供ConfigMap文件内容到指定路径使用
        configMap:
          name: example-volume-config      #ConfigMap中名称
          items:
          - key: log-script                #ConfigMap中的Key
            path: path/to/log-script       #指定目录下的一个相对路径path/to/log-script
      - name: nfs-client-root
        nfs:                               #挂载NFS类型储存
          server: 10.10.0.55               #NFS服务器地址
          path: /opt/public                #NFS中共享的路径
      - name: rbd-pvc
        persistentVolumeClaim:             #挂载PVC
          claimName: rbd-pvc1              #挂载已经申请的PVC
```

