作者：<https://github.com/daemon365/p/18592136>



---

* [为什么要做高可用](#_caption_0)
* [环境准备](#_caption_1)
* [安装](#_caption_2)
* [配置 keepalived](#_caption_3)
	+ [配置文件](#_caption0)
	+ [测试](#_caption1)
	+ [配置 haproxy](#_caption2)
* [安装 kubernetes 集群](#_caption_4)
* [测试](#_caption_5):[slowerssr加速器](https://slowerss.com)


## 为什么要做高可用


在生产环境中，kubernetes 集群中会多多个 master 节点，每个 master 节点上都会部署 kube\-apiserver 服务，实现高可用。但是 client 访问 kube\-apiserver 时，需要指定 ip 或者域名，这样会出现单点故障。官方推荐的做法是使用一个负载均衡器，将多个 kube\-apiserver 服务负载均衡，实现高可用，但很多时候我们是没有这个条件的。这时候就得想想办法了，比如 nignx 转发，但是 nginx 也是单点。域名的方式，但是这种方式生效时间较长，不太适合紧急情况。所以这里介绍一种使用 keepalived \+ haproxy 的方式实现 kube\-apiserver 的高可用。这是一共公用 IP 的方式，当主节点宕机时，VIP 会自动切换到备节点，实现高可用。


## 环境准备


* master1: 192\.168\.31\.203
* master2: 192\.168\.31\.34
* master3: 192\.168\.31\.46
* worker1: 192\.168\.31\.25
* VIP （虚拟IP）: 192\.168\.31\.230


## 安装



```


|  | sudo apt install keepalived haproxy |
| --- | --- |
|  |  |
|  | systemctl enable haproxy |
|  | systemctl restart haproxy |
|  |  |
|  | systemctl enable keepalived |
|  | # 没有配置会出现错误 不用管 |
|  | systemctl restart keepalived |


```

## 配置 keepalived


### 配置文件


编辑 keepalived 配置文件


编辑 /etc/keepalived/keepalived.conf


master1：



```


|  | # 健康检查 查看 haproxy 的进程在不在 |
| --- | --- |
|  | vrrp_script chk_haproxy { |
|  | script "killall -0 haproxy" |
|  | interval 2 # 多少秒教程一次 |
|  | weight 3 # 成功了优先级加多少 |
|  | } |
|  |  |
|  | vrrp_instance haproxy-vip { |
|  | state MASTER # MASTER / BACKUP 1 MASTER 2 BACKUP |
|  | priority 100 # 优先级 强的机器高一些 三台master 分别 100 99 98 |
|  | interface enp0s3     # 网卡名称 |
|  | virtual_router_id 51 # 路由 ip 默认就好 |
|  | advert_int 1 # keepalived 之间广播频率 秒 |
|  | authentication { |
|  | auth_type PASS |
|  | auth_pass test_k8s |
|  | } |
|  | unicast_src_ip 192.168.31.203 # 自己和其他 keepalived 通信地址 |
|  | unicast_peer { |
|  | 192.168.31.34                    # master2 的 IP 地址 |
|  | 192.168.31.46                     # master3 的 IP 地址 |
|  | } |
|  |  |
|  | virtual_ipaddress { |
|  | 192.168.31.230 # 这里必须和其他所有的ip 在一个局域网下 |
|  | } |
|  |  |
|  | track_script { |
|  | chk_haproxy |
|  | } |
|  | } |


```

master2：



```


|  | vrrp_script chk_haproxy { |
| --- | --- |
|  | script "killall -0 haproxy" |
|  | interval 2 |
|  | weight 3 |
|  | } |
|  |  |
|  | vrrp_instance haproxy-vip { |
|  | state BACKUP |
|  | priority 99 |
|  | interface enp0s3 |
|  | virtual_router_id 51 |
|  | advert_int 1 |
|  | authentication { |
|  | auth_type PASS |
|  | auth_pass test_k8s |
|  | } |
|  | unicast_src_ip 192.168.31.34 |
|  | unicast_peer { |
|  | 192.168.31.203 |
|  | 192.168.31.46 |
|  | } |
|  |  |
|  | virtual_ipaddress { |
|  | 192.168.31.230 |
|  | } |
|  |  |
|  | track_script { |
|  | chk_haproxy |
|  | } |
|  | } |
|  |  |


```

master3：



```


|  | vrrp_script chk_haproxy { |
| --- | --- |
|  | script "killall -0 haproxy" |
|  | interval 2 |
|  | weight 3 |
|  | } |
|  |  |
|  | vrrp_instance haproxy-vip { |
|  | state BACKUP |
|  | priority 98 |
|  | interface enp0s3 |
|  | virtual_router_id 51 |
|  | advert_int 1 |
|  | authentication { |
|  | auth_type PASS |
|  | auth_pass test_k8s |
|  | } |
|  | unicast_src_ip 192.168.31.46 |
|  | unicast_peer { |
|  | 192.168.31.203 |
|  | 192.168.31.34 |
|  | } |
|  |  |
|  | virtual_ipaddress { |
|  | 192.168.31.230 |
|  | } |
|  |  |
|  | track_script { |
|  | chk_haproxy |
|  | } |
|  | } |
|  |  |


```

### 测试


重启所有几点的 keepalived ， 虚拟 ip 会在节点 master 上，因为他的优先级高。



```


|  | # master 1 |
| --- | --- |
|  | ip a show enp0s3 |
|  | 2: enp0s3:  mtu 1500 qdisc fq_codel state UP group default qlen 1000 |
|  | link/ether 08:00:27:ca:59:86 brd ff:ff:ff:ff:ff:ff |
|  | inet 192.168.31.203/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s3 |
|  | valid_lft 41983sec preferred_lft 41983sec |
|  | inet 192.168.31.230/32 scope global enp0s3 |
|  | valid_lft forever preferred_lft forever |
|  | inet6 fe80::a00:27ff:feca:5986/64 scope link |
|  | valid_lft forever preferred_lft forever |


```

现在我们关掉 master1 的 haproxy 或者 keepalived



```


|  | systemctl stop haproxy |
| --- | --- |
|  | # 再查看网络信息 发现虚拟ip 没了 |
|  | ip a show enp0s3 |
|  | 2: enp0s3:  mtu 1500 qdisc fq_codel state UP group default qlen 1000 |
|  | link/ether 08:00:27:ca:59:86 brd ff:ff:ff:ff:ff:ff |
|  | inet 192.168.31.203/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s3 |
|  | valid_lft 41925sec preferred_lft 41925sec |
|  | inet6 fe80::a00:27ff:feca:5986/64 scope link |
|  | valid_lft forever preferred_lft forever |
|  |  |
|  | # 在优先级第二高的 master IP 上看下网络 |
|  | ip a show enp0s3 |
|  | 2: enp0s3:  mtu 1500 qdisc fq_codel state UP group default qlen 1000 |
|  | link/ether 08:00:27:11:af:4f brd ff:ff:ff:ff:ff:ff |
|  | inet 192.168.31.34/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s3 |
|  | valid_lft 41857sec preferred_lft 41857sec |
|  | inet 192.168.31.230/32 scope global enp0s3 |
|  | valid_lft forever preferred_lft forever |
|  | inet6 fe80::a00:27ff:fe11:af4f/64 scope link |
|  | valid_lft forever preferred_lft forever |
|  |  |
|  | # 启动 master1 的 haproxy ip就会回来 |


```

### 配置 haproxy


把 16443 端口的请求转发到 6443 端口 （3 master 的 kube\-apiserver 对外端口）


/etc/haproxy/haproxy.cfg



```


|  | global |
| --- | --- |
|  | log /dev/log  local0 warning |
|  | chroot      /var/lib/haproxy |
|  | pidfile     /var/run/haproxy.pid |
|  | maxconn     4000 |
|  | user        haproxy |
|  | group       haproxy |
|  | daemon |
|  |  |
|  | stats socket /var/lib/haproxy/stats |
|  |  |
|  | defaults |
|  | log global |
|  | option  httplog |
|  | option  dontlognull |
|  | timeout connect 5000 |
|  | timeout client 50000 |
|  | timeout server 50000 |
|  |  |
|  | frontend kube-apiserver |
|  | bind *:16443 |
|  | mode tcp |
|  | option tcplog |
|  | default_backend kube-apiserver |
|  |  |
|  | backend kube-apiserver |
|  | mode tcp |
|  | option tcp-check |
|  | balance roundrobin |
|  | default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100 |
|  | server kube-apiserver-1 192.168.31.203:6443 check |
|  | server kube-apiserver-2 192.168.31.34:6443 check |
|  | server kube-apiserver-3 192.168.31.46:6443 check |


```

## 安装 kubernetes 集群


master1



```


|  | kubeadm init --image-repository registry.aliyuncs.com/google_containers --control-plane-endpoint=192.168.31.230:16443 --v=10 |
| --- | --- |


```

master2 和 master3 加入集群



```


|  | kubeadm join 192.168.31.230:16443 --token rxblci.ddh60vl370wjgtn7         --discovery-token-ca-cert-hash sha256:d712016d5b8ba4ae5c4a1bda8b6ab1944c13a04757d2c488dd0aefcfd1af0157   --certificate-key    c398d693c6ce9b664634c9b670f013da3010580c00bd444caf7d0a5a81e803f5         --control-plane --v=10 |
| --- | --- |


```

worker 加入集群



```


|  | kubeadm join 192.168.31.230:16443 --token rxblci.ddh60vl370wjgtn7 \ |
| --- | --- |
|  | --discovery-token-ca-cert-hash sha256:d712016d5b8ba4ae5c4a1bda8b6ab1944c13a04757d2c488dd0aefcfd1af0157 |


```

查看集群状态



```


|  | kubectl get node |
| --- | --- |
|  | NAME      STATUS     ROLES           AGE     VERSION |
|  | master1   Ready      control-plane   21m     v1.28.2 |
|  | master2   Ready      control-plane   3m46s   v1.28.12 |
|  | master3   Ready      control-plane   2m12s   v1.28.12 |
|  | worker1   Ready                5s      v1.28.2 |


```

## 测试



```


|  | #  关闭 master1 的 kubelet 和 apiserver |
| --- | --- |
|  | systemctl stop kubelet |
|  | sudo kill -9 $(pgrep kube-apiserver) |
|  |  |
|  | kubectl get node |
|  | NAME      STATUS     ROLES           AGE     VERSION |
|  | master1   NotReady   control-plane   25m     v1.28.2 |
|  | master2   Ready      control-plane   7m40s   v1.28.12 |
|  | master3   Ready      control-plane   6m6s    v1.28.12 |
|  | worker1   Ready                3m59s   v1.28.2 |
|  |  |
|  |  |
|  | # 关闭 master1 的 haproxy |
|  | systemctl stop haproxy |
|  | root@master1:/home/zhy# kubectl get node |
|  | NAME      STATUS     ROLES           AGE     VERSION |
|  | master1   NotReady   control-plane   26m     v1.28.2 |
|  | master2   Ready      control-plane   9m12s   v1.28.12 |
|  | master3   Ready      control-plane   7m38s   v1.28.12 |
|  | worker1   Ready                5m31s   v1.28.2 |
|  |  |
|  | # 关闭 master2 的 keepalived |
|  | kubectl get node |
|  | NAME      STATUS     ROLES           AGE     VERSION |
|  | master1   NotReady   control-plane   28m     v1.28.2 |
|  | master2   Ready      control-plane   10m     v1.28.12 |
|  | master3   Ready      control-plane   9m12s   v1.28.12 |
|  | worker1   Ready                7m5s    v1.28.2 |
|  |  |
|  | # 可以看到 虚拟ip 跑到了 master3 上 |
|  | ip a show enp0s3 |
|  | 2: enp0s3:  mtu 1500 qdisc fq_codel state UP group default qlen 1000 |
|  | link/ether 08:00:27:f1:b5:ae brd ff:ff:ff:ff:ff:ff |
|  | inet 192.168.31.46/24 metric 100 brd 192.168.31.255 scope global dynamic enp0s3 |
|  | valid_lft 41021sec preferred_lft 41021sec |
|  | inet 192.168.31.230/32 scope global enp0s3 |
|  | valid_lft forever preferred_lft forever |
|  | inet6 fe80::a00:27ff:fef1:b5ae/64 scope link |
|  | valid_lft forever preferred_lft forever |


```

