### 相同宿主机下的dcoker之间通信

---

#### docker

---

docker的本质是进程，隔离的资源包括：网卡、回环设备、路由表和 iptables 规则，这些要素构成了一个进程(docker)发起和响应网络请求的基本环境。

#### host模式

---

--net=host 表示不开启Network Namespace，直接使用宿主机的网络。

#### bridge模式

---

- 容器拥有自己的ip和端口，这里只讨论此模式。

- docker会在宿主机上创建一个网桥docker0；当创建一个容器后，docker会生成一对veth pair设备，一个是容器内的eth0，一个挂载在宿主机的docker0上。

- Veth Pair 设备的特点是：它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的，并且从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网 卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。



**我们来创建两个容器**

```shell
docker run -itd --name cos1 centos:base /bin/bash -c 'while true;do echo 1;sleep 100;done'
docker run -itd --name cos2 centos:base /bin/bash -c 'while true;do echo 1;sleep 100;done'
```

宿主机查看docker0，发现绑定了两张虚拟网卡 

```shell
[root@kube-master src]# brctl show
docker0		8000.024250bc873d	no		veth4db29b2
                                        veth96c7e85
```

这两张网卡分别和两个容器的eth0组成了两对veth pair设备：veth4db29b2和cos1的eth0，veth96c7e85和cos2的eth0。

```shell
#cos1上查看：
[root@691e3302003c /]# route
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0

[root@691e3302003c /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 4  bytes 404 (404.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4  bytes 250 (250.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

从路由得知，所有的包都是从eth0出去的，对于目的地址为172.17.0.0/16的IP，匹配的是第二条路由规则。

所以，当从cos1上去ping cos2时

包从cos1上的eth0 -> docker0的veth4db29b2

docker0会进行ARP广播，然后发现cos2的MAC地址在veth96c7e85上，于是把这个包发给了veth96c7e85，然后进被cos2的eth0接收到了。

所以，当从cos1上去ping cos2时，网络链路如下：
cos1的eth0 -> docker0的veth4db29b2 -> docker0的veth96c7e85 -> cos2的eth0

响应顺序则正好相反。