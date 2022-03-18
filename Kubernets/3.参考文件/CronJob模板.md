# CronJob模板

```sh
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: 15job
spec:
  schedule: "0 5 * * *"                 #每天早上5点运行.cron表从左到右分别是:分钟,小时,每月日期,月,星期
  startingDeadlineSeconds: 500          #如果因为任何原因,时间超过了500秒而没启动任务,任务将不会运行并显示失败
  concurrencyPolicy: Forbid             #是否运行并行执行任务
  suspend: false                        #是否暂停任务
  jobTemplate:
    spec:
      template:                         #运行的Pod模板
        metadata:
          labels:
            app: 15job                  
        spec:
          restartPolicy: OnFailure      #重启策略,失败时重启
          hostNetwork: true             #共享节点网络命名空间
          hostPID: true                 #共享节点PID命名空间
          hostIPC: true                 #共享节点IPC命名空间
          nodeSelector:                 
            disktype: SSD               #选择运行到储存所在节点
          containers:
          - name: main
            image: luksa/batch-job
```

