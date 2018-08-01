### docker容器跨主机通信

前面了解了一下docker容器如何进行单主机内通信，在bridge模式下，docker容器联系外部依靠虚拟网桥进行桥接，以及nat进行。外部无法感觉到docker容器的存在。外部想要访问docker容器，需要在host上配置端口映射，配合iptables和nat将数据包转发给容器。



但是一般我们