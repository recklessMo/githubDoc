### docker网络的原理

http://cizixs.com/2016/06/01/docker-default-network 参考资料，讲的很好

1. 先建立一个概念，硬件交换机和软件交换机就是一个概念而已，首先要理解虚拟交换机，这样才能进一步的理解。虚拟交换机和硬件交换机是一样的，同样是决定数据包的交换，只是用软件来模拟而已。跳出固有的认识就可以了！同理，虚拟接口，虚拟网卡啥的也是一样的道理
2. 路由实际上是三层转发，默认linux系统来说，只会接收属于自己的数据包，但是如果接收到了不属于自己的数据包，如果打开了ip转发，那么就可以转发。这样linux系统就像一个路由器一样。通常是对于一个主机来说，如果有两张以上网卡，当一个网卡收到数据包之后，根据数据包的目的ip地址发到另外一张网卡上去。



主要是结合linux网络层面来了解一下docker不同的网络模式，了解一下不同的容器之间是怎么进行通信的！



默认情况下docker会在host上启动一个叫bridge0的bridge，可以认为这个bridge是一个虚拟的内核交换机，可以在不同的接口之间进行交换。在host上启动的容器都会连接到容器上，docker会从私有地址上选择一段来管理容器，比如172.17.0.1/16。一般都不在一个网段。



每次创建一个新的容器，docker都会创建一对interface，这对interface的特点就是从一端进入的网络流量只能从另外一端出来。这对接口一个对接在容器内部，比如容器的eth0，另外一个接口对接在本地host，类似veth**这种形式。这个veth接口会显示在host机器上，可以通过ip addr或者ifconfig命令来查看。



通过brctl show可以看到host机器上的bridge信息，可以看到bridge0上面挂了哪些接口。





