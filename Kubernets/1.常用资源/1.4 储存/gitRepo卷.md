# gitRepo卷

## 定义的作用

gitRepo卷基本上也是一个emptyDIr卷,通过pod启动时克隆Git仓库来填充数据.gitRepo卷创建完毕后,仓库并不会随着Git远程仓库更新,如果使用RS管理,删除pod将会新建pod,再次触发git clone可以获得最新文件.

一个典型的应用场景是存放网站的静态HTML文件,并创建一个包含web服务器容器和gitRepo卷的pod,当pod创建时,会拉取网站最新版本并启动web服务.不过每次有新版本需要删除pod才会更新.



## 使用gitRepo卷

使用一个Nginx容器和gitRepo卷:

```sh
[root@k8s-master ~]vi gitrepo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
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
    gitRepo:
      repository: https://github.com/luksa/kubia-wesite-example.git
      revision: master
      directory: .
[root@k8s-master ~]# kubectl create -f gitrepo-pod.yaml 
pod/gitrepo-volume-pod created
```

在gitRepo中需要revision指定master分支,同时将repo克隆到根目录.也可以指定文件夹名称.



## 私有仓库

gitRepo卷不能拉取私有Git repo仓库,例如需要账号密码验证的gitlab.如果需要添加支持,需要在pod中添加额外的sidecar容器,例如gitsync sidecar能对pod主容器操作同步仓库版本等.

gitRepo卷现在已被废弃,官方建议使用初始化容器将仓库中的数据复制到emptyDir储存卷上.

