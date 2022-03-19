# Ingress

## Ingress控制器工作

Ingress控制器运行一个反向代理服务器,类似于Nginx.根据集群中定义的Ingress, Service以及Endpoint资源来配置该控制器.所以需要订阅这些资源,然后每发生变化则更新代理服务器的配置.

尽管Ingress资源的定义指向一个Service,但Ingress控制器会直接将流量转到服务pod而不经过服务IP.当外部客户端通过Ingress控制器连接时,会对客户端IP进行保存,Service则做不到这一点.

