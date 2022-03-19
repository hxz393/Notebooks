# DNS

## DNS服务器工作

集群中所有pod默认配置使用集群内部DNS服务器,pod可以轻松地通过名称查询到集群内地服务,甚至是无头服务pod地IP地址.

DNS服务pod通过kube-dns服务对外暴露,DNS服务的IP地址在集群每个容器的/etc/reslv.conf文件中都有定义.

kube-dns pod利用API服务器的监控机制来订阅Service和Endpoint的变动,以及DNS记录的变更,使得客户端总能获取到最新的DNS信息.

