# CronJob

## 定义

可以创建CronJob资源来通过cron格式时间表,设置一个定时运行的Job任务.

下面简称CJ.K8s通过CJ中配置的Job模板创建Job资源.



## 创建CJ

创建一个每十五分钟运行一次的Job:

```sh
[root@server4-master ~]# vi kubia-cj.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: 15job
spec:
  schedule: "0,15,30,45 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: 15job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
[root@server4-master ~]# kubectl create -f kubia-cj.yaml 
[root@server4-master ~]# kubectl get cj
NAME    SCHEDULE             SUSPEND   ACTIVE   LAST SCHEDULE   AGE
15job   0,15,30,45 * * * *   False     0        <none>          16s
```

其中schedule段五个设置分别是: `分钟 小时 每月日期 月 星期`



## 超时设置

可以通过startingDeadlineSeconds字段来设置预定运行超时时间,例如不能超过预定时间15秒后运行:

```sh
[root@server4-master ~]# vi kubia-cj.yaml
spec:
  startingDeadlineSeconds: 15
```

不管任何原因,时间超过了15秒而没启动任务,任务将不会运行并显示失败.



## 其他设置

其他一些spec字段可嵌套使用字段:

- concurrencyPolicy: 并发执行策略,用于定义前一次作业运行尚未完成时是否以及如何运行后一次的作业.可选值Allow, Forbid或Replace.

- failedJobHistoryLimit: 为失败的任务执行保留的历史记录数,默认1.

- successfulJobsHistoryLimit: 为成功的任务执行保留的历史记录数,默认3.

- startingDeadlineSeconds: 设置超时时间,超时没完成任务会被记入错误历史记录.

- suspend: 是否挂起后续的任务执行,默认false,对运行中的作业不会产生影响.