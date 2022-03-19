# Job

## 定义

Job用来执行一个可完成的任务,进程终止之后不会再次启动.

在发生节点故障时,该节点上由Job管理的pod将重新安排到其他节点运行.如果节点本身异常退出(返回错误退出代码时),可以将Job配置为重新启动容器,由Job管理的pod会一直被重新安排直到任务完成为止.

Job使用api为batch/v1.需要明确地将重启策略设置为OnFailure或Nerver,防止容器在完成任务时重新启动.



## 创建Job

这里调用了一个运行120秒的进程然后退出.指定restartPolicy属性为OnFailure(默认为Always):

```sh
[root@server4-master ~]# vi kubia-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
[root@server4-master ~]# kubectl create -f kubia-job.yaml
```

当Job任务完成后状态显示为Completed:

```sh
[root@server4-master ~]# kubectl get jobs
NAME        COMPLETIONS   DURATION   AGE
batch-job   1/1           119s       2m59s
[root@server4-master ~]# kubectl get pod
NAME                 READY   STATUS      RESTARTS   AGE
batch-job--1-2ddkr   0/1     Completed   0          3m10s
```

出于查询日志的需求,完成任务的pod不会自动删除,可以通过删除创建的Job来一同删除.



## 多次运行

可以在Job中配置创建多个pod实例,设置completions和parallelism属性来以并行或串行方式运行.

顺序运行用于一个Job运行多次的场景,例如设置顺序运行五个pod,每个pod成功完成后工作才结束:

```sh
[root@server4-master ~]# vi kubia-job.yaml
spec:
  completions: 5
[root@server4-master ~]# kubectl create -f kubia-job.yaml
[root@server4-master ~]# kubectl get jobs
NAME        COMPLETIONS   DURATION   AGE
batch-job   0/5           12s        12s
```

设置并行运行需要多加一个parallelism参数设置并行启动pod数目:

```sh
[root@server4-master ~]# vi kubia-job.yaml
spec:
  completions: 5
  parallelism: 3
[root@server4-master ~]# kubectl delete job batch-job 
[root@server4-master ~]# kubectl create -f kubia-job.yaml
[root@server4-master ~]# kubectl get po
NAME                 READY   STATUS    RESTARTS   AGE
batch-job--1-5ghsj   1/1     Running   0          39s
batch-job--1-mctcz   1/1     Running   0          39s
batch-job--1-znpbq   1/1     Running   0          39s
```

Job的并行任务数同样可以通过scale命令来修改



## 限制条件

可以通过activeDeadlineSeconds属性来限制pod的运行时间.如果超时没完成,系统会尝试终止pod并标记失败.

还能配置Job被标记为失败前重试的次数,通过spec.backoffLimit字段,默认为6次.