#### 容器创建
----
* 自定义网络方式启动容器
```
docker-name="docker-with-macsec-01"
sudo docker run -idt --rm --net=none --name ${docker-name} ubuntu200403:nettools /bin/bash
```
* 获取容器网络空间
```
docker-pid-ns=$(docker inspect -f '{{.State.Pid}}' ${docker-name})
mkdir -p /var/run/netns
sudo ln -s /proc/${docker-pid-ns}/ns/net /var/run/netns/${docker-pid-ns}
```

#### 容器网络
----
*默认使用docker0网桥进行容器网络连接，也可以使用用户创建的网桥(包括ovs)进行连接*
*网络模型 docker-nic(macsec-nic)<---->veth-B<---->veth-A<---->bridge(docker0)*

* 创建veth pair
```
veth-pname="veth-01"
sudo ip link add ${veth-pname}-A type veth peer name ${veth-pname}-B
```
* 添加veth pair的A端到网桥
```
sudo brctl addif docker0 ${veth-pname}-A
sudo ip link set ${veth-pname}-A up
```
* 创建macsec网卡并与${veth-pname}-B绑定
```
macsec-nic-name="macsec-01"
sudo ip link add ${macsec-nic-name} link ${veth-pname}-B type macsec
sudo ip link set ${veth-pname}-B up
```
* 把创建的macsec网卡加入容器网络空间
```
macsec_nic_docker_ns_name="eth0"
macsec_nic_ip_address="172.192.192.100/24"
docker_ip_gateway="172.192.192.1"

sudo ip link set ${macsec-nic-name} netns ${docker-pid-ns}

sudo ip netns exec ${docker-pid-ns} ip link set dev ${macsec-nic-name} name ${macsec_nic_docker_ns_name}

sudo ip netns exec ${docker-pid-ns} ip link set ${macsec_nic_docker_ns_name} up

sudo ip netns exec ${docker-pid-ns} ip addr add ${macsec_nic_ip_address} dev ${macsec_nic_docker_ns_name}

sudo ip netns exec ${docker-pid-ns} ip route add default via ${docker_ip_gateway}
```
*至此容器${docker-name}已具备macsec网卡${macsec_nic_docker_ns_name}，同时此macsec网卡已接入到host的docker0网桥*
*但此macsec网卡还没有配置点对点的安全策略，还不能正常使用*


#### 配置macsec安全策略
----
*需要设置点对点加密SA H01<--->H02*
```
macsec_h01_mac_address="1a:06:07:ee:d9:b4"
macsec_h02_mac_address="92:36:87:57:b0:95"
macsec_h01_security_key=$(dd if=/dev/urandom count=16 bs=1 2>/dev/null | hexdump | cut -c 9- | tr -d ' \n')
macsec_h02_security_key=$(dd if=/dev/urandom count=16 bs=1 2>/dev/null | hexdump | cut -c 9- | tr -d ' \n')
```

```
//本地H01 SA配置
sudo ip netns exec ${docker-pid-ns} ip macsec add ${macsec_nic_docker_ns_name} rx port 1 address ${macsec_h02_mac_address}

sudo ip netns exec ${docker-pid-ns} ip macsec add ${macsec_nic_docker_ns_name} tx sa 0 pn 1 on key 00 ${macsec_h01_security_key}

sudo ip netns exec ${docker-pid-ns} ip macsec add ${macsec_nic_docker_ns_name} rx port 1 address ${macsec_h02_mac_address} sa 0 pn 1 on key 01 ${macsec_h02_security_key}
```

```
//对端H02 SA配置，对端可以是非容器网络
sudo ip macsec add ${macsec_nic_name} rx port 1 address ${macsec_h02_mac_address}

sudo ip macsec add ${macsec_nic_docker_ns_name} tx sa 0 pn 1 on key 01 ${macsec_h01_security_key}

ip macsec add ${macsec_nic_docker_ns_name} rx port 1 address ${macsec_h02_mac_address} sa 0 pn 1 on key 00 ${macsec_h02_security_key}
```
