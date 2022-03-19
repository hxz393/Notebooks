# emptyDir卷

## 定义

emptyDir卷是一个空目录,pod内的程序可以写入需要的任何文件,当删除pod时,卷的内容会丢失.主要用在pod中运行的容器之间临时共享文件.



## 使用emptyDir卷

下面使用Nginx作为服务器和Unix fortune命令来生成HTML内容,fortune命令每次运行都会输出一个随机引用,可以创建一个脚本每10秒运行一次,将其储存在index.html:

```sh
[root@server4-master ~]# vi fortune-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}                                 
[root@server4-master ~]# kubectl create -f fortune-pod.yaml 
pod/fortune created
```

第一个容器html-generator用来运行生成HTML文件到/var/htdocs中,同时把卷html挂载到目录/var/htdocs.

第二个容器web-server用来运行web服务,同时把卷html挂载到目录/usr/share/nginx/html,权限为只读.

最后定义一个名为html的单独emptyDir卷,供上面容器挂载.



## 查看pod状态

为了测试访问,可以直接设置端口转发来访问pod:

```sh
[root@server4-master ~]# kubectl port-forward fortune 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

新开一个终端,在本地通过curl来访问:

```sh
[root@server4-master ~]# curl 127.0.0.1:8080
Few things are harder to put up with than the annoyance of a good example.
                -- "Mark Twain, Pudd'nhead Wilson's Calendar"
```



## 指定储存介质

可以把emptyDir创建在tmfs,也就是内存中,空间受限于内存,但性能非常好,需要同时设置sizeLimit来限制使用.

下面创建一个Pod测试,包含tomcat和busybox两个容器.tomcat向挂载卷写入日志,busybox读取日志:

```sh
[root@server4-master ~]# vi empty-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
  namespace: default
spec:
  containers:
  - name: tomcat
    image: tomcat
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: app-logs
      mountPath: /usr/local/tomcat/logs
  - name: busybox
    image: busybox
    command: ["sh", "-c", "tail -f /logs/catalina*.log"]
    volumeMounts:
    - name: app-logs
      mountPath: /logs
  volumes:
  - name: app-logs
    emptyDir:
      medium: Memory
[root@server4-master ~]# kubectl create -f empty-pod.yaml 
pod/volume-pod created
```

查看日志:

```sh
[root@server4-master ~]# kubectl logs volume-pod -c busybox
14-Mar-2022 03:49:09.120 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.io.tmpdir=/usr/local/tomcat/temp
14-Mar-2022 03:49:09.128 INFO [main] org.apache.catalina.core.AprLifecycleListener.lifecycleEvent Loaded Apache Tomcat Native library [1.2.31] using APR version [1.7.0].
```

