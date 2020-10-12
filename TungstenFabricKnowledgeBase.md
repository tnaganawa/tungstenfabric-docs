

Table of Contents
=================

   * [Tungsten Fabric Knowledge Base](#tungsten-fabric-knowledge-base)
      * [vRouter internal](#vrouter-internal)
      * [control internal](#control-internal)
      * [config internal](#config-internal)
      * [config database internal](#config-database-internal)
      * [analytics internal](#analytics-internal)
      * [some configuration knobs which are not documented well](#some-configuration-knobs-which-are-not-documented-well)
         * [forwarding mode](#forwarding-mode)
         * [flood unknown unicast](#flood-unknown-unicast)
         * [allow tranisit](#allow-tranisit)
         * [multiple service chain](#multiple-service-chain)
      * [charm install](#charm-install)
      * [How to build tungsten fabric](#how-to-build-tungsten-fabric)
      * [vRouter scale test procedure](#vrouter-scale-test-procedure)
      * [multi kube-master deployment](#multi-kube-master-deployment)
      * [Nested kubernetes installation on openstack](#nested-kubernetes-installation-on-openstack)
      * [Tungsten fabric deployment on public cloud](#tungsten-fabric-deployment-on-public-cloud)
      * [erm-vpn](#erm-vpn)
      * [vRouter ml2 plugin](#vrouter-ml2-plugin)
      * [CentOS 8 installation procedure](#centos-8-installation-procedure)
      * [Random tungsten fabric patch (not tested)](#random-tungsten-fabric-patch-not-tested)


# Tungsten Fabric Knowledge Base

Miscellaneous topics for various deployment of tungsten fabric, which is not included in primer document.


## vRouter internal

### vhost0 device

when vRouter is firstly started, it will create vhost0 interface, and the ip and mac which originally assigned to physical interface, will be moved to vhost0.

So natural assumption is that, vhost0 is the vrouter itself, and it does arp response to the external fabric, and the traffic firstly go through vhost0, and enter the VM after that.
```
transit traffic:
 vm - vhost0 - eth0
self traffic:
 vhost0 - eth0
```

Actually, this is not the case.
 - As an illustration, when tcpdump is done against vhost0 for overlay traffic such as vxlan, it won't show some packets, and tcpdump against physical interface is needed for that purpose ..

Going through the source code, this diagram is the one actually used.
 - this document is also helpful to understand this: https://wiki.tungsten.io/display/TUN/Offloads?preview=%2F1409118%2F1409513%2FTungsten+Fabric+ParaVirt+Offload+Arch.DOCX
```
transit traffic:
 vm - (dp-core) - eth0
self traffic:
 vhost0 - (dp-core) - eth0
```

In this diagram, vhost0 is similar to irb in some bridge domain served by dp-core, and eth0 is one of the l2 interface in this bridge-domain.
 - In vrouter term, this state is called 'xconnect (cross-connect)', which is, in my understanding, similar to bridging: https://github.com/Juniper/contrail-vrouter/blob/master/dp-core/vr_interface.c#L231
 - bridge-domain is a similar concept with linux bridge, and it can have several physical l2-interfaces and one internal l3-interface.

So when eth0 firstly received arp requests from fabric, dp-core will return arp response, based on the mac address originally assigned to eth0.

Then other compute nodes will send some traffic, such as overlay traffic or self-traffic to that vRouter node.

When overlay traffic is used (it is identified by dp-core, based on udp port or gre header), dp-core will strip outer ip and label, and do vrf routing to the specific vm which the label indicates.
 - when l3 vxlan is used, it will do route lookup based on the routing table in l3 vrf
 - when mpls is used, the label itself identifies the final interface.

When it receives self traffic, dp-core will use hif_rx in vhost_tx (which, in turn, uses linux function netif_rx, with skb as an argument) to send the traffic to the linux interface, which is vhost0, on vRouter node.
 - https://github.com/Juniper/contrail-vrouter/blob/master/dp-core/vr_interface.c#L813
 - https://github.com/Juniper/contrail-vrouter/blob/master/linux/vr_host_interface.c#L2380
 - https://github.com/Juniper/contrail-vrouter/blob/master/linux/vr_host_interface.c#L228

So for rx / tx for self-traffic, packets always go through dp-core, and for transit traffic, it won't go through vhost0.

### skb to vr_packet

Linux network stack uses sk_buff as the memory store of packet data.

In dp-core, instead, vr_packet is used, so it is an interesting subject how they are converted to each other.

For that purpose, vp_os_packet function is used.
 - https://github.com/Juniper/contrail-vrouter/blob/master/include/vr_linux.h#L10
```
static inline struct sk_buff *
vp_os_packet(struct vr_packet *pkt)
{
    return CONTAINER_OF(cb, struct sk_buff, pkt);
}
```

So vr_packet is actually defined in some location in skb struct (sk_buff->cb, which is a member variable used by some application), so skb and vr_packet can be converted by pointer operation.


One note, since cb's maximum size is 48 bytes, vr_packet can't be larger than that. Some comments describe this.
https://github.com/Juniper/contrail-vrouter/blob/master/include/vr_packet.h#L195-L198
```
/*
 * NOTE: Please do not add any more fields without ensuring
 * that the size is <= 48 bytes in 64 bit systems.
 */
```

### linux interfaces created by vRouter

Several interfaces are created when vrouter-agent container is firstly started, and it is actually not deleted even if vrouter-agent is stopped.

For what purpose it is used is an interesting subject.

To summarize that, vif interface in vrouter.ko is always tied to corresponding linux netdevice,
so creating some vrouter interface with such as vif --create, will also create linux netdevice, which can be seen from such as ip link or ls /sys/class/net.

One illustlation comes from 'ip tuntap list'.
```
[root@ip-172-31-12-55 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: ens3    inet6 fe80::46c:bff:fec8:dd64/64 scope link \       valid_lft forever preferred_lft forever
3: docker0    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0\       valid_lft forever preferred_lft forever
16: vhost0    inet 172.31.12.55/20 brd 172.31.15.255 scope global dynamic vhost0\       valid_lft 3118sec preferred_lft 3118sec
16: vhost0    inet6 fe80::46c:bff:fec8:dd64/64 scope link \       valid_lft forever preferred_lft forever
17: pkt0    inet6 fe80::5094:6cff:fefb:42f7/64 scope link \       valid_lft forever preferred_lft forever
[root@ip-172-31-12-55 ~]# ip tuntap list
pkt0: tap
[root@ip-172-31-12-55 ~]#
```
, so pkt0 is actually a tap device from linux's point of view.

In a sense, vif command will tigh vrouter interface to some linux netdevice, such as tapxxxx-xxxx, which will be created by such as nova-vif-driver, to make the packet which go through that device to be received by dp-core.

So when such as CNI found a tap device, which is connected to containers, it will send vrouter-api, which internally create vif, with the same name as the tap device, so that the packets which entered tap device will be forwarded to vRouter (dp-core).


There are some special devices, which will be created when vrouter-agent is started, namely vhost0, pkt0, pkt1, pkt2, pkt3.

As described before, vhost0 is similar to irb interface of dp-core, so it will receive the packet to the vRouter node itself, after dp-core routing has finished.

Since vrouter-agent container will create /etc/sysconfig/network-scripts/{ifup-vhost,ifdown-vhost} when it starts, it can be directly controlled by ifup / ifdown, which internally type vif --add vhost0, it can be directly created and deleted from command line.
 - https://github.com/Juniper/contrail-container-builder/blob/master/containers/vrouter/base/network-functions-vrouter-kernel#L41

pkt1, pkt2, pkt3, are interfaces defined in linux_pkt_dev_alloc, in vrouter_linux_init, which is the module_init of vrouter.ko.
 - https://github.com/Juniper/contrail-vrouter/blob/master/linux/vr_host_interface.c#L2485

```
linux/vrouter_mod.c
 module_init(vrouter_linux_init);

static int
linux_pkt_dev_alloc(void)
{
    if (pkt_gro_dev == NULL) {
        pkt_gro_dev = linux_pkt_dev_init("pkt1", &pkt_gro_dev_setup,
                                         &pkt_gro_dev_rx_handler);
        if (pkt_gro_dev == NULL) {
            vr_module_error(-ENOMEM, __FUNCTION__, __LINE__, 0);
            return -ENOMEM;
        }
    }

    if (pkt_l2_gro_dev == NULL) {
        pkt_l2_gro_dev = linux_pkt_dev_init("pkt3", &pkt_l2_gro_dev_setup,
                                         &pkt_gro_dev_rx_handler);
        if (pkt_l2_gro_dev == NULL) {
            vr_module_error(-ENOMEM, __FUNCTION__, __LINE__, 0);
            return -ENOMEM;
        }
    }

    if (pkt_rps_dev == NULL) {
        pkt_rps_dev = linux_pkt_dev_init("pkt2", &pkt_rps_dev_setup,
                                        &pkt_rps_dev_rx_handler);
        if (pkt_rps_dev == NULL) {
            vr_module_error(-ENOMEM, __FUNCTION__, __LINE__, 0);
            return -ENOMEM;
        }
    }

    return 0;
}
```

It uses some GRO and RPS feature, which is important to make kernel vRouter's performance better.
 - They are initialized empty net_device_ops and random ethernet addr.

```
linux/vr_host_interface.c


/*
 * pkt_rps_dev_ops - netdevice operations on RPS packet device. Currently,
 * no operations are needed, but an empty structure is required to
 * register the device.
 *
 */
static struct net_device_ops pkt_rps_dev_ops;

(snip)

/*
 * pkt_rps_dev_setup - fill in the relevant fields of the RPS packet device
 */
static void
pkt_rps_dev_setup(struct net_device *dev)
{
    /*
     * Initializing the interfaces with basic parameters to setup address
     * families.
     */
    random_ether_addr(dev->dev_addr);
    dev->addr_len = ETH_ALEN;

    dev->hard_header_len = ETH_HLEN;

    dev->type = ARPHRD_VOID;
    dev->netdev_ops = &pkt_rps_dev_ops;
    dev->mtu = 65535;

    return;
}
```

pkt0 is slightly different, and it is used to send the packets from dp-core to vrouter-agent.

It is actually created when vrouter-agent is firstly started, by the request from vrouter-agent, to create tap device to communicate with vrouter-agent.
 - https://github.com/Juniper/contrail-controller/blob/master/src/vnsw/agent/contrail/contrail_agent_init.cc#L89
 - https://github.com/Juniper/contrail-controller/blob/master/src/vnsw/agent/oper/interface.cc#L626
 - https://github.com/Juniper/contrail-vrouter/blob/master/dp-core/vr_interface.c#L669

So if packets are sent from dp-core to that interface, vrouter-agent will receive that, to handle that packet internally (arp, dhcp, ... are handled in that way)


As one more illustration of this behavior, I'll add ip -o addr, ip link, vif --list result, when modprobe vrouter, ifup vhost0, vrouter-agent start are done.

```
# docker-compose -f /etc/contrail/vrouter/docker-compose.yaml down
# ifdown vhost0
# modprobe vrouter

[root@ip-172-31-12-55 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: ens3    inet 172.31.12.55/20 brd 172.31.15.255 scope global dynamic ens3\       valid_lft 3561sec preferred_lft 3561sec
2: ens3    inet6 fe80::46c:bff:fec8:dd64/64 scope link \       valid_lft forever preferred_lft forever
3: docker0    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0\       valid_lft forever preferred_lft forever
[root@ip-172-31-12-55 ~]# 

[root@ip-172-31-12-55 ~]# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 06:6c:0b:c8:dd:64 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:34:e8:c3:14 brd ff:ff:ff:ff:ff:ff
9: pkt1: <> mtu 65535 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/void be:f9:01:0e:4d:38 brd 00:00:00:00:00:00
10: pkt3: <> mtu 65535 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/void 46:f8:5c:cb:79:8e brd 00:00:00:00:00:00
11: pkt2: <> mtu 65535 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/void a2:b0:40:5c:03:d4 brd 00:00:00:00:00:00
[root@ip-172-31-12-55 ~]#
[root@ip-172-31-12-55 ~]# vif --list
Vrouter Interface Table

Flags: P=Policy, X=Cross Connect, S=Service Chain, Mr=Receive Mirror
       Mt=Transmit Mirror, Tc=Transmit Checksum Offload, L3=Layer 3, L2=Layer 2
       D=DHCP, Vp=Vhost Physical, Pr=Promiscuous, Vnt=Native Vlan Tagged
       Mnp=No MAC Proxy, Dpdk=DPDK PMD Interface, Rfl=Receive Filtering Offload, Mon=Interface is Monitored
       Uuf=Unknown Unicast Flood, Vof=VLAN insert/strip offload, Df=Drop New Flows, L=MAC Learning Enabled
       Proxy=MAC Requests Proxied Always, Er=Etree Root, Mn=Mirror without Vlan Tag, HbsL=HBS Left Intf
       HbsR=HBS Right Intf, Ig=Igmp Trap Enabled

vif0/4350   OS: pkt3
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

vif0/4351   OS: pkt1
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

[root@ip-172-31-12-55 ~]# 


# ifup vhost0

[root@ip-172-31-12-55 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: ens3    inet6 fe80::46c:bff:fec8:dd64/64 scope link \       valid_lft forever preferred_lft forever
3: docker0    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0\       valid_lft forever preferred_lft forever
12: vhost0    inet 172.31.12.55/20 brd 172.31.15.255 scope global dynamic vhost0\       valid_lft 3594sec preferred_lft 3594sec
12: vhost0    inet6 fe80::46c:bff:fec8:dd64/64 scope link \       valid_lft forever preferred_lft forever
[root@ip-172-31-12-55 ~]# 
[root@ip-172-31-12-55 ~]# 
[root@ip-172-31-12-55 ~]# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 06:6c:0b:c8:dd:64 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:34:e8:c3:14 brd ff:ff:ff:ff:ff:ff
9: pkt1: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void be:f9:01:0e:4d:38 brd 00:00:00:00:00:00
10: pkt3: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void 46:f8:5c:cb:79:8e brd 00:00:00:00:00:00
11: pkt2: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void a2:b0:40:5c:03:d4 brd 00:00:00:00:00:00
12: vhost0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 06:6c:0b:c8:dd:64 brd ff:ff:ff:ff:ff:ff
[root@ip-172-31-12-55 ~]# 

[root@ip-172-31-12-55 ~]# vif --list
Vrouter Interface Table

Flags: P=Policy, X=Cross Connect, S=Service Chain, Mr=Receive Mirror
       Mt=Transmit Mirror, Tc=Transmit Checksum Offload, L3=Layer 3, L2=Layer 2
       D=DHCP, Vp=Vhost Physical, Pr=Promiscuous, Vnt=Native Vlan Tagged
       Mnp=No MAC Proxy, Dpdk=DPDK PMD Interface, Rfl=Receive Filtering Offload, Mon=Interface is Monitored
       Uuf=Unknown Unicast Flood, Vof=VLAN insert/strip offload, Df=Drop New Flows, L=MAC Learning Enabled
       Proxy=MAC Requests Proxied Always, Er=Etree Root, Mn=Mirror without Vlan Tag, HbsL=HBS Left Intf
       HbsR=HBS Right Intf, Ig=Igmp Trap Enabled

vif0/2      OS: ens3 (Speed 10000, Duplex 1)
            Type:Physical HWaddr:06:6c:0b:c8:dd:64 IPaddr:0.0.0.0
            Vrf:0 Mcast Vrf:65535 Flags:XTcL3L2Vp QOS:0 Ref:1
            RX packets:54  bytes:13325 errors:0
            TX packets:39  bytes:4452 errors:0
            Drops:0

vif0/16     OS: vhost0
            Type:Host HWaddr:06:6c:0b:c8:dd:64 IPaddr:0.0.0.0
            Vrf:0 Mcast Vrf:65535 Flags:XL3L2 QOS:0 Ref:1
            RX packets:39  bytes:4452 errors:0
            TX packets:54  bytes:13325 errors:0
            Drops:0

vif0/4350   OS: pkt3
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

vif0/4351   OS: pkt1
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

[root@ip-172-31-12-55 ~]# 

# docker-compose -f /etc/contrail/vrouter/docker-compose.yaml up -d

[root@ip-172-31-12-55 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: ens3    inet6 fe80::46c:bff:fec8:dd64/64 scope link \       valid_lft forever preferred_lft forever
3: docker0    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0\       valid_lft forever preferred_lft forever
16: vhost0    inet 172.31.12.55/20 brd 172.31.15.255 scope global dynamic vhost0\       valid_lft 3552sec preferred_lft 3552sec
16: vhost0    inet6 fe80::46c:bff:fec8:dd64/64 scope link \       valid_lft forever preferred_lft forever
17: pkt0    inet6 fe80::5094:6cff:fefb:42f7/64 scope link \       valid_lft forever preferred_lft forever
[root@ip-172-31-12-55 ~]# 
[root@ip-172-31-12-55 ~]# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 06:6c:0b:c8:dd:64 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:34:e8:c3:14 brd ff:ff:ff:ff:ff:ff
13: pkt1: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void 36:72:98:97:9b:31 brd 00:00:00:00:00:00
14: pkt3: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void 92:aa:52:e8:d5:c5 brd 00:00:00:00:00:00
15: pkt2: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void 42:b2:46:73:3d:6c brd 00:00:00:00:00:00
16: vhost0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 06:6c:0b:c8:dd:64 brd ff:ff:ff:ff:ff:ff
17: pkt0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 52:94:6c:fb:42:f7 brd ff:ff:ff:ff:ff:ff
[root@ip-172-31-12-55 ~]# 
[root@ip-172-31-12-55 ~]# vif --list
Vrouter Interface Table

Flags: P=Policy, X=Cross Connect, S=Service Chain, Mr=Receive Mirror
       Mt=Transmit Mirror, Tc=Transmit Checksum Offload, L3=Layer 3, L2=Layer 2
       D=DHCP, Vp=Vhost Physical, Pr=Promiscuous, Vnt=Native Vlan Tagged
       Mnp=No MAC Proxy, Dpdk=DPDK PMD Interface, Rfl=Receive Filtering Offload, Mon=Interface is Monitored
       Uuf=Unknown Unicast Flood, Vof=VLAN insert/strip offload, Df=Drop New Flows, L=MAC Learning Enabled
       Proxy=MAC Requests Proxied Always, Er=Etree Root, Mn=Mirror without Vlan Tag, HbsL=HBS Left Intf
       HbsR=HBS Right Intf, Ig=Igmp Trap Enabled

vif0/0      OS: ens3 (Speed 10000, Duplex 1) NH: 4
            Type:Physical HWaddr:06:6c:0b:c8:dd:64 IPaddr:0.0.0.0
            Vrf:0 Mcast Vrf:65535 Flags:TcL3L2VpEr QOS:-1 Ref:7
            RX packets:165  bytes:97837 errors:0
            TX packets:156  bytes:124911 errors:0
            Drops:0

vif0/1      OS: vhost0 NH: 5
            Type:Host HWaddr:06:6c:0b:c8:dd:64 IPaddr:172.31.12.55
            Vrf:0 Mcast Vrf:65535 Flags:PL3DEr QOS:-1 Ref:8
            RX packets:159  bytes:125878 errors:0
            TX packets:192  bytes:98971 errors:0
            Drops:7

vif0/2      OS: pkt0
            Type:Agent HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3Er QOS:-1 Ref:3
            RX packets:31  bytes:2666 errors:0
            TX packets:34  bytes:13535 errors:0
            Drops:0

vif0/4350   OS: pkt3
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

vif0/4351   OS: pkt1
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

[root@ip-172-31-12-55 ~]# 
```

### vRouter API

When an orchestrator (openstack or kubernetes) specifies a compute node which will serve specific virtual-machine, it will serve information of that node through nova-api or kube-apiserver, and vrouter-agent will create its vif based on that information.

To do that, vrouter-agent has vRouter API (TCP/9091), and nova-vif-driver and vRouter CNI will call this, based on the information served by an orcherstrator and config-api.
 - https://github.com/tungstenfabric/tf-controller/blob/master/src/vnsw/agent/port_ipc/rest_server.cc
 - https://github.com/tungstenfabric/tf-nova-vif-driver/
 - https://github.com/tungstenfabric/tf-controller/blob/master/src/container/cni/contrail/vrouter.go#L29

When this API is called, it will create tap device and vif device, to forward packets from VMs to dp-core.

It also associates some specific virtual-machine objects to that virtual-router, and ifmap objects associated to that virtual-machine will be downloaded to that virtual-router. 
 - https://github.com/tungstenfabric/tf-controller/blob/master/src/ifmap/README.md

After that, that virtual-router will start to serve bgp route for that virtual-machine.


### application-policy-set and security-group

vRouter supports application tag and security-group without using iptables.

To do this, it maintains flow identication logic, which is similar to some stateful firewall implementation, in vrouter-agent.
 - the key is that it is not maintained in vrouter.ko

When vrouter.ko sees the packets whose 5-tuple doesn't have an entry in its table (flow table, which is a hash table inside vrouter.ko), it will send 5-tuple of that packet to vrouter-agent, through pkt0 tap.
 - https://github.com/Juniper/contrail-controller/wiki/Flow-processing

Since vrouter-agent have all the info about routing and security-policy, when it received that packet, it will identify the next-hop and flow action, to return the packet to vrouter.ko and vrouter.ko will create a flow 
entry.
 - https://github.com/tungstenfabric/tf-controller/blob/master/src/vnsw/agent/pkt/flow_entry.cc#L2482-L2512

When identifying flow action after route lookup, vrouter-agent will check all the bgp enhanced community associated to that prefix, to determine flow action for this packet.

```
For example, when deny policy is configured between label=vm1 and label=vm1-1, vRouter will receive this ACL.
 - src_type: tags and dst_type: tags clarifies that it is for fw-policy between tags, and 6, 7 is the tag IDs for label=vm1 and label=vm1-1


# ./ist.py vr acl 0534aa3f-2b64-4b97-ac16-fa908bdffefd
+--------------------------------------+--------------------------------------+-------------+
| uuid                                 | name                                 | dynamic_acl |
+--------------------------------------+--------------------------------------+-------------+
| 0534aa3f-2b64-4b97-ac16-fa908bdffefd | default-policy-management:fw-policy1 | false       |
+--------------------------------------+--------------------------------------+-------------+

# ./ist.py vr acl 0534aa3f-2b64-4b97-ac16-fa908bdffefd -f text
AclSandeshData
  uuid: 0534aa3f-2b64-4b97-ac16-fa908bdffefd
  dynamic_acl: false
  entries
      AclEntrySandeshData
        ace_id: 0
        rule_type: Terminal
        src: 6
        dst: 7
        src_port_l
            SandeshRange
              min: 0
              max: 65535
        dst_port_l
            SandeshRange
              min: 0
              max: 65535
        proto_l
            SandeshRange
              min: 1
              max: 1
        action_l
            ActionStr
              action: deny
        src_type: tags
        dst_type: tags
        uuid: 45e2f437-87f2-4457-9ff5-610a0cd0458d
      AclEntrySandeshData
        ace_id: 0
        rule_type: Terminal
        src: 7
        dst: 6
        src_port_l
            SandeshRange
              min: 0
              max: 65535
        dst_port_l
            SandeshRange
              min: 0
              max: 65535
        proto_l
            SandeshRange
              min: 1
              max: 1
        action_l
            ActionStr
              action: deny
        src_type: tags
        dst_type: tags
        uuid: 45e2f437-87f2-4457-9ff5-610a0cd0458d
	(snip)
  name: default-policy-management:fw-policy1



Those ACLs are assigned to the virtual-machine-interface which label=vm1 is associated to.
  - vmi_tag_list and policy_set_acl_list describe the detail


# ./ist.py vr intf tap4b603c36-2c
+-------+----------------+--------+-------------------+---------------+---------------+---------+-----------------------------+
| index | name           | active | mac_addr          | ip_addr       | mdata_ip_addr | vm_name | vn_name                     |
+-------+----------------+--------+-------------------+---------------+---------------+---------+-----------------------------+
| 4     | tap4b603c36-2c | Active | 02:4b:60:3c:36:2c | 192.168.100.3 | 169.254.0.4   | vm1     | default-domain:admin:testvn |
+-------+----------------+--------+-------------------+---------------+---------------+---------+-----------------------------+

# ./ist.py vr intf tap4b603c36-2c -f text
ItfSandeshData
  index: 4
  name: tap4b603c36-2c
  uuid: 4b603c36-2cb7-44a7-8d06-81009876eee6
  vrf_name: default-domain:admin:testvn:testvn
  active: Active
  ipv4_active: Active
  l2_active: L2 Active
  ip6_active: Ipv6 Inactive < no-ipv6-addr  >
  health_check_active: Active
  dhcp_service: Enable
  dns_service: Enable
  type: vport
  label: 23
  l2_label: 27
  vxlan_id: 9
  vn_name: default-domain:admin:testvn
  vm_uuid: b678a2aa-8a81-49cb-974a-bc2457e07685
  vm_name: vm1
  ip_addr: 192.168.100.3
  mac_addr: 02:4b:60:3c:36:2c
  policy: Enable
  fip_list
  mdata_ip_addr: 169.254.0.4
  service_vlan_list
  os_ifindex: 11
  fabric_port: NotFabricPort
  alloc_linklocal_ip: LL-Enable
  analyzer_name
  config_name: default-domain:admin:4b603c36-2cb7-44a7-8d06-81009876eee6
  sg_uuid_list
      VmIntfSgUuid
        sg_uuid: 178b6839-f732-4dc8-913e-98e44a6c70a4
  static_route_list
  vm_project_uuid: 07778a3d-0c46-4c4b-9054-462c23122b98
  admin_state: Enabled
  flow_key_idx: 24
  allowed_address_pair_list
  ip6_addr: ::
  local_preference: 0
  tx_vlan_id: -1
  rx_vlan_id: -1
  parent_interface
  subnet: --NA--
  sub_type: Tap
  vrf_assign_acl_uuid: --NA--
  vmi_type: Virtual Machine
  transport: Ethernet
  logical_interface_uuid: 00000000-0000-0000-0000-000000000000
  flood_unknown_unicast: false
  physical_device
  physical_interface
  fixed_ip4_list
      192.168.100.3
  fixed_ip6_list
  fat_flow_list
  metadata_ip_active: Active
  service_health_check_ip: 0.0.0.0
  alias_ip_list
  drop_new_flows: false
  bridge_domain_list
  vmi_tag_list
      VmiTagData
        name: label=vm1
        id: 6
        application_policy_set_list
  policy_set_acl_list
      0534aa3f-2b64-4b97-ac16-fa908bdffefd
  slo_list
  vhostuser_mode: 0
  si_other_end_vmi: 00000000-0000-0000-0000-000000000000
  cfg_igmp_enable: false
  igmp_enabled: false
  max_flows: 0
  policy_set_fwaas_list
  bond_interface_list



The prefix to that virtual-machine-interface also has that tag as an enhanced coummunity.
 - tag_list identifies the value associated to that prefix.


# ./ist.py vr vrf testvn
+------------------------------------+---------+---------+---------+-----------+----------+-----------------------------+
| name                               | ucindex | mcindex | brindex | evpnindex | vxlan_id | vn                          |
+------------------------------------+---------+---------+---------+-----------+----------+-----------------------------+
| default-domain:admin:testvn:testvn | 5       | 5       | 5       | 5         | 9        | default-domain:admin:testvn |
+------------------------------------+---------+---------+---------+-----------+----------+-----------------------------+

# ./ist.py vr route -v 5 192.168.100.3
0.0.0.0/0
    [192.168.122.52] pref:200
     to 2:d:44:5d:b4:14 via veth5bc41975-f, assigned_label:33, nh_index:38 , nh_type:interface, nh_policy:enabled, active_label:33, vxlan_id:0
    [192.168.122.53] pref:200
     to 2:d:44:5d:b4:14 via veth5bc41975-f, assigned_label:33, nh_index:38 , nh_type:interface, nh_policy:enabled, active_label:33, vxlan_id:0
192.168.100.0/24
    [Local] pref:100
     nh_index:1 , nh_type:discard, nh_policy:disabled, active_label:-1, vxlan_id:0
192.168.100.3/32
    [192.168.122.52] pref:200
     to 2:4b:60:3c:36:2c via tap4b603c36-2c, assigned_label:23, nh_index:24 , nh_type:interface, nh_policy:enabled, active_label:23, vxlan_id:0
    [192.168.122.53] pref:200
     to 2:4b:60:3c:36:2c via tap4b603c36-2c, assigned_label:23, nh_index:24 , nh_type:interface, nh_policy:enabled, active_label:23, vxlan_id:0
    [LocalVmPort] pref:200
     to 2:4b:60:3c:36:2c via tap4b603c36-2c, assigned_label:23, nh_index:24 , nh_type:interface, nh_policy:enabled, active_label:23, vxlan_id:0
    [INET-EVPN] pref:100
     nh_index:0 , nh_type:None, nh_policy:, active_label:-1, vxlan_id:0


# ./ist.py vr route -v 5 192.168.100.3 -r
  (snip)
RouteUcSandeshData
  src_ip: 192.168.100.3
  src_plen: 32
  src_vrf: default-domain:admin:testvn:testvn
  path_list
      PathSandeshData
        nh
          NhSandeshData
            type: interface
            ref_count: 8
            valid: true
            policy: enabled
            itf: tap4b603c36-2c
            mac: 2:4b:60:3c:36:2c
            mcast: disabled
            nh_index: 24
            vxlan_flag: false
            intf_flags: 1
            isid: 0
            learning_enabled: false
            etree_leaf: false
            layer2_control_word: false
            crypt_all_traffic: false
            crypt_path_available: false
            crypt_interface
        label: 23
        vxlan_id: 0
        peer: 192.168.122.52
        dest_vn_list
            default-domain:admin:testvn
        unresolved: false
        sg_list
            8000002
        supported_tunnel_type: MPLSoGRE MPLSoUDP
        active_tunnel_type: MPLSoUDP
        stale: false
        path_preference_data
          PathPreferenceSandeshData
            sequence: 1
            preference: 200
            ecmp: false
        active_label: 23
        ecmp_hashing_fields: l3-source-address,l3-destination-address,l4-protocol,l4-source-port,l4-destination-port,
        communities
        peer_sequence_number: 1
        etree_leaf: false
        layer2_control_word: false
        tag_list
            6
        inactive: false
        evpn_dest_vn_list
     (snip)
  ipam_subnet_route: false
  ipam_host_route: true
  proxy_arp: false
  multicast: false
  intf_route_type: interface
```

Since each vRouter receives all the prefix from other vRouters with its associated bgp enhanced community, each vRouter can determine if it can allow / drop that packets when it firstly enters one of vRouters.


## control internal
### ifmap-server deprecation

After R4.0, ifmap-server is deprecated, and currently control nodes directly received config information from cassandra.
 - https://github.com/Juniper/contrail-specs/blob/master/deprecating-discovery-4.0.md

Having said that, internally, it still uses ifmap structs, to store topology data for vrf, interface, logical-router, etc.

To pick the data directly from cassandra, some changes are made for ifmap clients, which is used by control.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1632470

Originally, ifmap client contains a lot of logic to pick data from ifmap-server, but currently it contains only one logic, to get json file from cassandra, and fill ifmap struct with that data.
 - https://github.com/Juniper/contrail-controller/tree/R2002/src/ifmap/client
 - https://github.com/Juniper/contrail-controller/tree/R3.2/src/ifmap/client

So it now uses ifmap as the struct for internal use, not as a wire protocol.

### difference between named and dns
contrail-dns, and contrail-named is a different process, and it is actually used with different purpose.

contrail-dns has similar feature with contrail-control, and it serves vDNS info with XMPP, and vRouter will do some DNS tasks based on that input.
 - https://github.com/Juniper/contrail-controller/tree/master/src/dns
 - https://github.com/Juniper/contrail-controller/wiki/DNS-and-IPAM

contrail-named, actually, doesn't use XMPP, and it uses ISC bind to serve DNS data, and it is used for external DNS query to vDNS entry.
 - https://github.com/Juniper/contrail-container-builder/blob/master/containers/controller/control/named/entrypoint.sh#L10

## config internal
### CRUD operation REST API and msgbus update
Config-api will serve REST API for CRUD operation of each config objects, such as virtual-network, network-policy, ...

To serve this, it dynamically creates URL, based on schema file.
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/api-server/vnc_cfg_api_server/vnc_cfg_api_server.py#L1835-L1891
 - _generate_resource_crud_methods and _generate_resource_crud_uri creates generic methods and URL

Default behavior for this methods are to do cassandra update, and rabbitmq exchange is also filled with some message, for other processes, such as schema-transformer, svc-monitor, device-manager, to consume.
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/api-server/vnc_cfg_api_server/vnc_db.py#L1595-L1606

```
    def dbe_create(self, obj_type, obj_uuid, obj_dict):
        (ok, result) = self._object_db.object_create(obj_type, obj_uuid,
                                                     obj_dict)

        if ok:
            # publish to msgbus
            self._msgbus.dbe_publish('CREATE', obj_type, obj_uuid,
                                     obj_dict['fq_name'], obj_dict=obj_dict)
            self._dbe_publish_update_implicit(obj_type, result)

        return (ok, result)
    # end dbe_create
```

Other tasks such as checking input data, or fill that with default value, will be done by pre_dbe_create, or post_dbe_create (create can be delete, update, read etc) and it is defined per resource.
 - https://github.com/Juniper/contrail-controller/tree/master/src/config/api-server/vnc_cfg_api_server/resources

### dependency_tracker

Dependency tracker is used by schema-transformer, svc-monitor, and device-manager, to consume amqp message from config-api, and recursively evaluated the objects referenced by the one updated.
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/common/cfgm_common/dependency_tracker.py

Internally, if reaction_map, which is a python dict with object name updated and other object name which need to evaluated, contains a key for the object included in amqp message, it will start evaluating that object.

For example, if virtual-machine-interface is updated,
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/schema-transformer/schema_transformer/to_bgp.py#L92-L94

```
        'virtual_machine_interface': {
            'self': ['virtual_machine', 'port_tuple', 'virtual_network',
                     'bgp_as_a_service'],
```

it will also evaluate virtual-machine, port-tuple, virtual-network, and bgp-as-a-service, if it has a reference with the virtual-machine-interface, which is originally updated.

### number of config push by device-manager

device-manager pushes device configuration, when some config-database objects are updated.

One interesting question is that when objects are updated several times, how many config push will occur.

To maintain this state, device-manager uses a queue (size=1).
 - https://github.com/tungstenfabric/tf-controller/blob/master/src/config/device-manager/device_manager/db.py#L538

So config objects are updated several times, device-manager starts pushing device configuration based on the first AMQP message, and the second AMQP messages will be stored in that queue, the all the other messages are dropped from that queue.

So even if several config object updates occurred in a short period, device config push will occur only two times.

## config database internal

### to read config_db_uuid keyspace contents

When cassandra contents are seen by cqlsh (for example cql> select * from config_db_uuid.obj_fq_name_table;), it returns some output which is not human-readable.

Key is that config-api internally uses pycassa's ColumnFamily (https://github.com/pycassa/pycassa#basic-usage), which is similar to ORM mapper for cassandra.
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/common/cfgm_common/cassandra_driver_thrift.py#L251-L259

```
    def _cassandra_init_conn_pools(self):
(snip)
            for cf_name in cf_dict:
                cf_kwargs = cf_dict[cf_name].get('cf_args', {})
                self._cf_dict[cf_name] = ColumnFamily(
                    pool,
                    cf_name,
                    read_consistency_level=ConsistencyLevel.QUORUM,
                    write_consistency_level=ConsistencyLevel.QUORUM,
                    dict_class=dict,
                    **cf_kwargs)
```

To read this, json file created by backup / restore procedure is convinient,
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#backup-and-restore

although that result is mostly similar to config-api's HTTP GET output ..


### zookeeper usage in tungsten fabric config database

Since it is not easy to calculate the next integer by cassandra, tungsten fabric uses zookeeper for that purpose.
https://stackoverflow.com/questions/53702288/is-increment-integer-in-cassandra-possible-in-some-cases

Data for that is in various zookeeper path. To investigate, those command can be used.

```
docker exec -it config_database_zookeeper_1 bash
  ./bin/zkCli.sh -server config-database-ip

[zk: 172.31.12.209(CONNECTED) 0] ls /
[analytics-discovery-, api-server-election, device-manager, fq-name-to-uuid, id, schema-transformer, svc-monitor, zookeeper]
[zk: 172.31.12.209(CONNECTED) 1] ls /id
[bgp, security-groups, tags, virtual-networks]
[zk: 172.31.12.209(CONNECTED) 2] ls /id/bgp
[route-targets]
[zk: 172.31.12.209(CONNECTED) 3] ls /id/bgp/route-targets
[type0]
[zk: 172.31.12.209(CONNECTED) 4] ls /id/bgp/route-targets/type0
[0008000000, 0008000001, 0008000002, 0008000003, 0008000004, 0008000005, 0008000006]
[zk: 172.31.12.209(CONNECTED) 5] 

[zk: 172.31.12.209(CONNECTED) 11] get /id/bgp/route-targets/type0/0008000000
default-domain:default-project:default-virtual-network:default-virtual-network
[zk: 172.31.12.209(CONNECTED) 12] 
```

Backup script also can be used to dump all the Zookeeper data.
https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#backup-and-restore


## analytics internal
### redis, cassandra and kafka
Analytics has several backend databases, most notabliy, redis and cassandra, and optionally kafka, if alarmgen is also installed.

Those databases are separately updated by collector, when sandesh update from such as vRouter, control has reached to this process.

When it is received, it reaches ruleeng.cc,
 - https://github.com/tungstenfabric/tf-analytics/blob/master/contrail-collector/ruleeng.cc#L900

and do some tasks based on the sandesh data type received.
 - If that is UVE, fill redis and kafka, and if cassandra is installed, also fill stat tables of this db.
```
    // First publish to redis and kafka
    if (uveproc) handle_uve_publish(parent, vmsgp, db, header, db_cb);
    // Check if the message needs to be dropped
    if (db && db->DropMessage(header, vmsgp)) {
        return true;
    }

    if (db) {
        // 1. make entry in OBJECT_VALUE_TABLE if needed
        // 2. get object-type:name{1-6}
        handle_object_log(parent, vmsgp, db, header, &object_names, db_cb);

        // Insert into the message table
        db->MessageTableInsert(vmsgp, object_names, db_cb);

        if (uveproc) handle_uve_statistics(parent, vmsgp, db, header, db_cb);

        handle_session_object(parent, db, header, db_cb);
    }
```

So redis and kafka will handle UVE only, and when cassandra is not installed, all the data other than UVE is not imported to analytics databases.

### uveupdate.lua

There are some lua files in collector directory, which is used to update redis, when UVE reached to the collector.
 - https://github.com/tungstenfabric/tf-analytics/blob/master/contrail-collector/uveupdate.lua

Internally, it is converted to cpp file by xxd command when it is compiled.
 - https://github.com/tungstenfabric/tf-analytics/blob/master/contrail-collector/SConscript#L155-L160

```
def RedisLuaBuild(env, scr_name):
  env.Command('%s_lua.cpp' % scr_name ,'%s.lua' % scr_name,\
                  '(cd %s ; xxd -i %s.lua > ../../../%s/%s_lua.cpp)' %
              (Dir('#src/contrail-analytics/contrail-collector').path,
               scr_name, Dir('.').path, scr_name))
  env.Depends('redis_processor_vizd.cc','%s_lua.cpp' % scr_name)
```

### ContrailConfig UVE

In analytics UVE, ContrailConfig is created to show the configuration which is maintained in config-database.
 - this value can be used for some alarms, which are to see configuration is the same with the value sent from such as vRouter, to check the sanity of that installation.

Although most of UVEs are sent from such as vRouter itself, ContrailConfig is sent from config-api to analytics.
 - https://github.com/tungstenfabric/tf-controller/blob/master/src/config/api-server/vnc_cfg_api_server/vnc_db.py#L1806-L1811

So this value should be in the UVE entry, even if vrouter-agent is not running.

## some configuration knobs which are not documented well

### forwarding mode

vRouter has several forwarding mode.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1471637

By default, it will use l2 / l3 mode. L3 mode and L2 mode also have some usecase.


### flood unknown unicast

This knob is used when l2 BMS connectivity is used.
- https://bugs.launchpad.net/juniperopenstack/+bug/1424523

By default, vRouter will drop unknown unicast, since controller knows all mac address of VMs, although it is not the case when l2 BMS is used.

This knob need to be enabled in such case.


### allow tranisit

This knob is used with service-chain feature.
 - https://bugs.launchpad.net/opencontrail/+bug/1365277

When VM1 - VN1 - VNF1 - VN2 - VNF2 - VN3 - VM2 is created and VN1-VN2, VN2-VN3 service-chain is configured, by default, VM1 can't ping VM2, since ServiceChain prefix is not transitive.

When this knob is enabled in VN2, prefix in VN1 will be imported to VN3 and vice versa, so VM1 can ping to VM3.

### multiple service chain

I actually never tried this knob.

Detail is described in this url.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1505023
 
## charm install

Tungsten Fabric can also be installed by juju charm.
- bionic and openstack queens is used, 4 nodes (juju node, openstack controller, openstack compute, tunsten-fabric controller)

```
# apt update
# snap install --classic juju
# juju add-cloud

Select cloud type: manual
Enter a name for your manual cloud: manual-cloud-1
Enter the controller's hostname or IP address: (juju node's ip)

# ssh-keygen
# cd .ssh
# cat id_rsa.pub >> authorized_keys
# cd
# ssh-copy-id (other nodes' ip)

# juju bootstrap manual-cloud-1

# git clone https://github.com/Juniper/contrail-charms -b R5

# juju add-machine ssh:root@(openstack-controller ip)
# juju add-machine ssh:root@(openstack-compute ip)
# juju add-machine ssh:root@(TungstenFabric-controller ip)

# vi set-juju.sh
juju deploy ntp
juju deploy rabbitmq-server --to lxd:0
juju deploy percona-cluster mysql --config root-password=contrail123 --config max-connections=1500 --to lxd:0
juju deploy openstack-dashboard --to lxd:0
juju deploy nova-cloud-controller --config console-access-protocol=novnc --config network-manager=Neutron --to lxd:0
juju deploy neutron-api --config manage-neutron-plugin-legacy-mode=false --config neutron-security-groups=true --to lxd:0
juju deploy glance --to lxd:0
juju deploy keystone --config admin-password=contrail123 --config admin-role=admin --to lxd:0

juju deploy nova-compute --config ./nova-compute-config.yaml --to 1

CHARMS_DIRECTORY=/root
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-keystone-auth --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-controller --config auth-mode=rbac --config cassandra-minimum-diskgb=4 --config cassandra-jvm-extra-opts="-Xms1g -Xmx2g" --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-analyticsdb --config cassandra-minimum-diskgb=4 --config cassandra-jvm-extra-opts="-Xms1g -Xmx2g" --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-analytics --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-openstack
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-agent

juju expose openstack-dashboard
juju expose nova-cloud-controller
juju expose neutron-api
juju expose glance
juju expose keystone

juju expose contrail-controller
juju expose contrail-analytics

juju add-relation keystone:shared-db mysql:shared-db
juju add-relation glance:shared-db mysql:shared-db
juju add-relation keystone:identity-service glance:identity-service
juju add-relation nova-cloud-controller:image-service glance:image-service
juju add-relation nova-cloud-controller:identity-service keystone:identity-service
juju add-relation nova-cloud-controller:cloud-compute nova-compute:cloud-compute
juju add-relation nova-compute:image-service glance:image-service
juju add-relation nova-compute:amqp rabbitmq-server:amqp
juju add-relation nova-cloud-controller:shared-db mysql:shared-db
juju add-relation nova-cloud-controller:amqp rabbitmq-server:amqp
juju add-relation openstack-dashboard:identity-service keystone

juju add-relation neutron-api:shared-db mysql:shared-db
juju add-relation neutron-api:neutron-api nova-cloud-controller:neutron-api
juju add-relation neutron-api:identity-service keystone:identity-service
juju add-relation neutron-api:amqp rabbitmq-server:amqp

juju add-relation contrail-controller ntp
juju add-relation nova-compute:juju-info ntp:juju-info

juju add-relation contrail-controller contrail-keystone-auth
juju add-relation contrail-keystone-auth keystone
juju add-relation contrail-controller contrail-analytics
juju add-relation contrail-controller contrail-analyticsdb
juju add-relation contrail-analytics contrail-analyticsdb

juju add-relation contrail-openstack neutron-api
juju add-relation contrail-openstack nova-compute
juju add-relation contrail-openstack contrail-controller

juju add-relation contrail-agent:juju-info nova-compute:juju-info
juju add-relation contrail-agent contrail-controller

# vi nova-compute-config.yaml 
nova-compute:
    virt-type: qemu 
    enable-resize: True
    enable-live-migration: True
    migration-auth-type: ssh

# bash set-juju.sh

(to check status, it takes 20 minutes for every application to be active)
# juju status
# tail -f /var/log/juju/*log | grep -v -w DEBUG
```

To make this work, two points are to be cared of.
1. Since juju internally use LXD and its own subnets, at least tungsten fabric nodes need to have some static route to that subnet (if AWS is used, VPC's route table can be used, Source/Dest Check also need to be disabled)
2. Since LXD by default won't allow docker to run, it needs to be allowed by lxc config.

```
juju ssh 0
  sudo su -
    lxc list
    lxc config set juju-cb8047-0-lxd-4 security.nesting true
    lxc config show juju-cb8047-0-lxd-4
    lxc restart juju-cb8047-0-lxd-4
```

## How to build tungsten fabric

This repo's readme mostly worked well.
https://github.com/Juniper/contrail-dev-env

```
yum -y install docker git
git clone https://github.com/Juniper/contrail-dev-env
cd contrail-dev-env
./startup.sh
docker exec -it contrail-developer-sandbox bash

cd /root/contrail-dev-env
yum -y remove python-devel ## it is needed to resolve dependency issue
make sync
make fetch_packages
make setup
make dep
```

To build all the modules, this command can be used (it takes 1-2 hours, depending on machine power)
```
make rpm
make containers
```

To build more specific modules, those command also can be used.
One notice is that rpm-contrail is a big package itself, and can't decomposed more .. (controller, vrouter etc is included)
```
make list
make rpm-contrail

make list-containers
make container-general-base
make container-base
make container-kubernetes_kube-manager

- those make targets are included from this file:
 /root/contrail/tools/packages/Makefile
 https://github.com/Juniper/contrail-packages/blob/master/Makefile
```

To build only vrouter.ko, this command worked for me.
```
build:
cd /root/contrail
scons --opt=production --kernel-dir=/lib/modules/3.10.0-1062.el7.x86_64/build build-kmodule

clean:
cd /root/contrail/vrouter
make KERNELDIR=/lib/modules/3.10.0-1062.el7.x86_64/build clean
```

Note: When kernel-devel package from other distrubution (I tried one from centos 8 and amazon linux 2) is installed, it can also be specified as kernel-dir.

For example, this command created vrouter.ko for centos 8.2.
 - it can be manually loaded by insmod command.
```
# rpm -ivh --nodeps kernel-devel-4.18.0-147.8.1.el8_1.x86_64.rpm
# scons --opt=production --kernel-dir=/usr/src/kernels/4.18.0-147.8.1.el8_1.x86_64/ build-kmodule
```

## vRouter scale test procedure

Since GCP allowed me to start up to 5k nodes :), this procedure is described primarily for this platform.
 - Having said that, the same procedure also can be used with AWS

First target was 2k vRouter nodes, but as far as I tried, it is not the maximum, and more might be done with more control nodes or CPU/MEM are added.

In GCP, VPC can be created with several subnets, so control plane nodes have been assigned 172.16.1.0/24, and vRouter nodes have 10.0.0.0/9. (default subnets has /12, and up to 4k nodes can be used)

By default, not all the instances can have global ip, so Cloud NAT need to be defined for vRouter nodes, to access the internet. (I assigned global IPs to control plane nodes, since number of nodes won't be that much)

All the nodes are created by instance-group, with auto-scaling disabled, and fixed number assigned.
All the nodes are preemptive enabled for lower cost ($0.01/1hr (n1-standard-1) for vRouter, and $0.64/1hr (n1-standard-64) for control plane nodes)

Overall procedure is decribed as follows.
1. Setup control/config x 5, analytics x 3, kube-master x 1 with this procedure.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#2-tungstenfabric-up-and-running

It takes up to 30 minutes.

2002-latest is used. JVM_EXTRA_OPTS: "-Xms128m -Xmx20g" is set.

One point to be added, XMPP_KEEPALIVE_SECONDS determines XMPP scalability, and I set this 3
So after vRouter node failure, it takes 9 seconds for control to recognize it. (by default it is set 10/30)
I suppose this is a moderate choice for IaaS usecase, but if this value needs to be lower, more CPU would be needed.

For the later use, virtual-network vn1 (10.1.0.0/12, l2/l3) is also created.
 - https://github.com/tnaganawa/tungctl/blob/master/samples.yaml#L3

2. One kube-master will be set up with this procedure.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#kubeadm

It takes up to 20 minutes.

For cni.yaml, this url is used.
 - https://raw.githubusercontent.com/tnaganawa/tungstenfabric-docs/master/multi-kube-master-deployment-cni-tungsten-fabric.yaml

XMPP_KEEPALIVE_SECONDS: "3" is added to env.

Because of this vRouter issue on GCP,
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricKnowledgeBase.md#vrouters-on-gce-cannot-reach-other-nodes-in-the-same-subnet

vrouter-agent container is patched, and yaml need to be changed.
```
      - name: contrail-vrouter-agent
        image: "tnaganawa/contrail-vrouter-agent:2002-latest"  ### this line is changed
```

set-label.sh and kubectl apply -f cni.yaml is done at this time.

3. start vRouter nodes, and dump the ips with this command.
```
(for GCP)
gcloud --format="value(networkInterfaces[0].networkIP)" compute instances list

(for AWS, this command can be used)
aws ec2 describe-instances --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text | tr '\t' '\n' 
```

It takes about 10-20 minutes.


4. Install kubernetes on vRouter nodes, and wait for vRouter nodes are installed by that.
```
(/tmp/aaa.pem is the secret key for GCP)
sudo yum -y install epel-release
sudo yum -y install parallel
sudo su - -c "ulimit -n 8192; su - centos"
cat all.txt | parallel -j3500 scp -i /tmp/aaa.pem -o StrictHostKeyChecking=no install-k8s-packages.sh centos@{}:/tmp
cat all.txt | parallel -j3500 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} chmod 755 /tmp/install-k8s-packages.sh
cat all.txt | parallel -j3500 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} sudo /tmp/install-k8s-packages.sh

### this command needs to be up to 200 parallel execution, since without that, it leads to timeout of kubeadm join
cat all.txt | parallel -j200 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} sudo kubeadm join 172.16.1.x:6443 --token we70in.mvy0yu0hnxb6kxip --discovery-token-ca-cert-hash sha256:13cf52534ab14ee1f4dc561de746e95bc7684f2a0355cb82eebdbd5b1e9f3634
```

kubeadm join takes about 20-30 minutes.
vRouter installation takes about 40-50 minutes. (based on Cloud NAT performance, to add docker registry in VPC would improve docker pull time).


5. after that, first-containers.yaml can be created with replica: 2000, and ping between containers can be checked.
 To see BUM behavior, vn1 also can be used with containers with 2k replicas.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/8-leaves-contrail-config.txt#L99

Container creation will take up to 15-20 minutes.


```
[config results]

2200 instances are created, and 2188 kube workers are available.
(some are restarted and not available, since those instance are preemptive VM)

[centos@instance-group-1-srwq ~]$ kubectl get node | wc -l
2188
[centos@instance-group-1-srwq ~]$ 

When vRouters are installed, some more nodes are rebooted, and 2140 vRouters become available.

Every 10.0s: kubectl get pod --all-namespaces | grep contrail | grep Running | wc -l     Sun Feb 16 17:25:16 2020
2140

After start creating 2k containers, 15 minutes is needed before 2k containers are up.

Every 5.0s: kubectl get pod -n myns11 | grep Running | wc -l                                                             Sun Feb 16 17:43:06 2020
1927


ping between containers works fine:

$ kubectl get pod -n myns11
(snip)
vn11-deployment-68565f967b-zxgl4   1/1     Running             0          15m   10.0.6.0     instance-group-3-bqv4   <none>           <none>
vn11-deployment-68565f967b-zz2f8   1/1     Running             0          15m   10.0.6.16    instance-group-2-ffdq   <none>           <none>
vn11-deployment-68565f967b-zz8fk   1/1     Running             0          16m   10.0.1.61    instance-group-4-cpb8   <none>           <none>
vn11-deployment-68565f967b-zzkdk   1/1     Running             0          16m   10.0.2.244   instance-group-3-pkrq   <none>           <none>
vn11-deployment-68565f967b-zzqb7   0/1     ContainerCreating   0          15m   <none>       instance-group-4-f5nw   <none>           <none>
vn11-deployment-68565f967b-zzt52   1/1     Running             0          15m   10.0.5.175   instance-group-3-slkw   <none>           <none>
vn11-deployment-68565f967b-zztd6   1/1     Running             0          15m   10.0.7.154   instance-group-4-skzk   <none>           <none>

[centos@instance-group-1-srwq ~]$ kubectl exec -it -n myns11 vn11-deployment-68565f967b-zzkdk sh
/ # 
/ # 
/ # ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
36: eth0@if37: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue \    link/ether 02:fd:53:2d:ea:50 brd ff:ff:ff:ff:ff:ff
36: eth0    inet 10.0.2.244/12 scope global eth0\       valid_lft forever preferred_lft forever
36: eth0    inet6 fe80::e416:e7ff:fed3:9cc5/64 scope link \       valid_lft forever preferred_lft forever
/ # ping 10.0.1.61
PING 10.0.1.61 (10.0.1.61): 56 data bytes
64 bytes from 10.0.1.61: seq=0 ttl=64 time=3.635 ms
64 bytes from 10.0.1.61: seq=1 ttl=64 time=0.474 ms
^C
--- 10.0.1.61 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.474/2.054/3.635 ms
/ #

There are some XMPP flap .. it might be caused by CPU spike by config-api, or some effect of preemptive VM.
It needs to be investigated further with separating config and control.
(most of other 2k vRouter nodes won't experience XMPP flap though)

(venv) [centos@instance-group-1-h26k ~]$ ./contrail-introspect-cli/ist.py ctr nei -t XMPP -c flap_count | grep -v -w 0
+------------+
| flap_count |
+------------+
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
| 1          |
+------------+
(venv) [centos@instance-group-1-h26k ~]$ 


[BUM tree]

Send two multicast packets.

/ # ping 224.0.0.1
PING 224.0.0.1 (224.0.0.1): 56 data bytes
^C
--- 224.0.0.1 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
/ # 
/ # ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
36: eth0@if37: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue \    link/ether 02:fd:53:2d:ea:50 brd ff:ff:ff:ff:ff:ff
36: eth0    inet 10.0.2.244/12 scope global eth0\       valid_lft forever preferred_lft forever
36: eth0    inet6 fe80::e416:e7ff:fed3:9cc5/64 scope link \       valid_lft forever preferred_lft forever
/ # 

That container is on this node.

(venv) [centos@instance-group-1-h26k ~]$ ping instance-group-3-pkrq
PING instance-group-3-pkrq.asia-northeast1-b.c.stellar-perigee-161412.internal (10.0.3.211) 56(84) bytes of data.
64 bytes from instance-group-3-pkrq.asia-northeast1-b.c.stellar-perigee-161412.internal (10.0.3.211): icmp_seq=1 ttl=63 time=1.46 ms


It sends overlay packet to some other endpoints (not all 2k nodes),

[root@instance-group-3-pkrq ~]# tcpdump -nn -i eth0 udp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
17:48:51.501718 IP 10.0.3.211.57333 > 10.0.0.212.6635: UDP, length 142
17:48:52.501900 IP 10.0.3.211.57333 > 10.0.0.212.6635: UDP, length 142

and it eventually reach other containers, going through Edge Replicate tree.

[root@instance-group-4-cpb8 ~]# tcpdump -nn -i eth0 udp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
17:48:51.517306 IP 10.0.1.198.58095 > 10.0.5.244.6635: UDP, length 142
17:48:52.504484 IP 10.0.1.198.58095 > 10.0.5.244.6635: UDP, length 142




[resource usage]

controller:
CPU usage is moderate and bound by contrail-control process.
If more vRouter nodes need to be added, more controller nodes can be added.
 - separating config and control also should help to reach further stability

top - 17:45:28 up  2:21,  2 users,  load average: 7.35, 12.16, 16.33
Tasks: 577 total,   1 running, 576 sleeping,   0 stopped,   0 zombie
%Cpu(s): 14.9 us,  4.2 sy,  0.0 ni, 80.8 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem : 24745379+total, 22992752+free, 13091060 used,  4435200 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 23311113+avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
23930 1999      20   0 9871.3m   5.5g  14896 S  1013  2.3   1029:04 contrail-contro
23998 1999      20   0 5778768 317660  12364 S 289.1  0.1 320:18.42 contrail-dns                        
13434 polkitd   20   0   33.6g 163288   4968 S   3.3  0.1  32:04.85 beam.smp                                
26696 1999      20   0  829768 184940   6628 S   2.3  0.1   0:22.14 node                                 
 9838 polkitd   20   0   25.4g   2.1g  15276 S   1.3  0.9  45:18.75 java                                     
 1012 root      20   0       0      0      0 S   0.3  0.0   0:00.26 kworker/18:1                       
 6293 root      20   0 3388824  50576  12600 S   0.3  0.0   0:34.39 docker-containe                                                 
 9912 centos    20   0   38.0g 417304  12572 S   0.3  0.2   0:25.30 java                                                 
16621 1999      20   0  735328 377212   7252 S   0.3  0.2  23:27.40 contrail-api                                      
22289 root      20   0       0      0      0 S   0.3  0.0   0:00.04 kworker/16:2                                          
24024 root      20   0  259648  41992   5064 S   0.3  0.0   0:28.81 contrail-nodemg                                                  
48459 centos    20   0  160328   2708   1536 R   0.3  0.0   0:00.33 top                                                             
61029 root      20   0       0      0      0 S   0.3  0.0   0:00.09 kworker/4:2                                                      
    1 root      20   0  193680   6780   4180 S   0.0  0.0   0:02.86 systemd                                                        
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.03 kthreadd         

[centos@instance-group-1-rc34 ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           235G         12G        219G        9.8M        3.9G        222G
Swap:            0B          0B          0B
[centos@instance-group-1-rc34 ~]$ 
[centos@instance-group-1-rc34 ~]$ df -h .
/dev/sda1         10G  5.1G  5.0G   51% /
[centos@instance-group-1-rc34 ~]$ 
   


analytics:
CPU usage is moderate and bound by contrail-collector process.
If more vRouter nodes need to be added, more analytics nodes can be added.

top - 17:45:59 up  2:21,  1 user,  load average: 0.84, 2.57, 4.24
Tasks: 515 total,   1 running, 514 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.3 us,  1.3 sy,  0.0 ni, 95.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 24745379+total, 24193969+free,  3741324 used,  1772760 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 24246134+avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                       
 6334 1999      20   0 7604200 958288  10860 S 327.8  0.4 493:31.11 contrail-collec          
 4904 polkitd   20   0  297424 271444   1676 S  14.6  0.1  10:42.34 redis-server           
 4110 root      20   0 3462120  95156  34660 S   1.0  0.0   1:21.32 dockerd                     
    9 root      20   0       0      0      0 S   0.3  0.0   0:04.81 rcu_sched                      
   29 root      20   0       0      0      0 S   0.3  0.0   0:00.05 ksoftirqd/4           
 8553 centos    20   0  160308   2608   1536 R   0.3  0.0   0:00.07 top                
    1 root      20   0  193564   6656   4180 S   0.0  0.0   0:02.77 systemd                 
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.03 kthreadd             
    4 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H               
    5 root      20   0       0      0      0 S   0.0  0.0   0:00.77 kworker/u128:0         
    6 root      20   0       0      0      0 S   0.0  0.0   0:00.17 ksoftirqd/0     
    7 root      rt   0       0      0      0 S   0.0  0.0   0:00.38 migration/0    

[centos@instance-group-1-n4c7 ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           235G        3.6G        230G        8.9M        1.7G        231G
Swap:            0B          0B          0B
[centos@instance-group-1-n4c7 ~]$ 
[centos@instance-group-1-n4c7 ~]$ df -h .
/dev/sda1         10G  3.1G  6.9G   32% /
[centos@instance-group-1-n4c7 ~]$ 




kube-master:
CPU usage is small.

top - 17:46:18 up  2:22,  2 users,  load average: 0.92, 1.32, 2.08
Tasks: 556 total,   1 running, 555 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.2 us,  0.5 sy,  0.0 ni, 98.2 id,  0.1 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem : 24745379+total, 23662128+free,  7557744 used,  3274752 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 23852964+avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                      
 5177 root      20   0 3605852   3.1g  41800 S  78.1  1.3 198:42.92 kube-apiserver                                                               
 5222 root      20   0   10.3g 633316 410812 S  55.0  0.3 109:52.52 etcd                                                                         
 5198 root      20   0  846948 651668  31284 S   8.3  0.3 169:03.71 kube-controller                                                              
 5549 root      20   0 4753664  96528  34260 S   5.0  0.0   4:45.54 kubelet                                                                      
 3493 root      20   0 4759508  71864  16040 S   0.7  0.0   1:52.67 dockerd-current                                                              
 5197 root      20   0  500800 307056  17724 S   0.7  0.1   6:43.91 kube-scheduler                                                               
19933 centos    20   0  160340   2648   1528 R   0.7  0.0   0:00.07 top                                                                          
 1083 root       0 -20       0      0      0 S   0.3  0.0   0:20.19 kworker/0:1H                                                                 
35229 root      20   0       0      0      0 S   0.3  0.0   0:15.08 kworker/0:2                                                                  
    1 root      20   0  193808   6884   4212 S   0.0  0.0   0:03.59 systemd                                                                      
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.03 kthreadd                                                                     
    4 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H                                                                 
    5 root      20   0       0      0      0 S   0.0  0.0   0:00.55 kworker/u128:0                                                               
    6 root      20   0       0      0      0 S   0.0  0.0   0:01.51 ksoftirqd/0      

[centos@instance-group-1-srwq ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           235G         15G        217G        121M        3.2G        219G
Swap:            0B          0B          0B
[centos@instance-group-1-srwq ~]$ 

[centos@instance-group-1-srwq ~]$ df -h /
/dev/sda1         10G  4.6G  5.5G   46% /
[centos@instance-group-1-srwq ~]$ 

[centos@instance-group-1-srwq ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           235G         15G        217G        121M        3.2G        219G
Swap:            0B          0B          0B
[centos@instance-group-1-srwq ~]$ 

[centos@instance-group-1-srwq ~]$ df -h /
/dev/sda1         10G  4.6G  5.5G   46% /
[centos@instance-group-1-srwq ~]$ 

```




## multi kube-master deployment

3 tungsten fabric controller nodes: m3.xlarge (4 vcpu) -> c3.4xlarge (16 vcpu) # since schema-transformer needed cpu resource for acl calculation, I need to add resource  
100 kube-master, 800 workers: m3.medium

tf-controller installation and first-containers.yaml is the same with this url
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#2-tungstenfabric-up-and-running

Ami is also the same (ami-3185744e), but kernel version is updated by yum -y update kernel (converted to image, and it is used to start instances)

/tmp/aaa.pem is the keypair, specified in ec2 instances

cni.yaml is attached:
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/multi-kube-master-deployment-cni-tungsten-fabric.yaml

```
(commands are typed on one of tungsten fabric controller nodes)
yum -y install epel-release
yum -y install parallel

aws ec2 describe-instances --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text | tr '\t' '\n' > /tmp/all.txt
head -n 100 /tmp/all.txt > masters.txt
tail -n 800 /tmp/all.txt > workers.txt

ulimit -n 4096
cat all.txt | parallel -j1000 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} id
cat all.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
 -
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' ssh -i /tmp/aaa.pem centos@{2} sudo kubeadm init --token aaaaaa.aaaabbbbccccdddd --ignore-preflight-errors=NumCPU --pod-network-cidr=10.32.{1}.0/24 --service-cidr=10.96.{1}.0/24 --service-dns-domain=cluster{1}.local
-
vi assign-kube-master.py
computenodes=8
with open ('masters.txt') as aaa:
 with open ('workers.txt') as bbb:
  for masternode in aaa.read().rstrip().split('\n'):
   for i in range (computenodes):
    tmp=bbb.readline().rstrip()
    print ("{}\t{}".format(masternode, tmp))
 -
cat masters.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo cp /etc/kubernetes/admin.conf /tmp/admin.conf
cat masters.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo chmod 644 /tmp/admin.conf
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' scp -i /tmp/aaa.pem centos@{2}:/tmp/admin.conf kubeconfig-{1}
-
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' kubectl --kubeconfig=kubeconfig-{1} get node
-
cat -n join.txt | parallel -j1000 -a - --colsep '\t' ssh -i /tmp/aaa.pem centos@{3} sudo kubeadm join {2}:6443 --token aaaaaa.aaaabbbbccccdddd --discovery-token-unsafe-skip-ca-verification
-
(modify controller-ip in cni-tungsten-fabric.yaml)
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' cp cni-tungsten-fabric.yaml cni-{1}.yaml
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' sed -i -e "s/k8s2/k8s{1}/" -e "s/10.32.2/10.32.{1}/" -e "s/10.64.2/10.64.{1}/" -e "s/10.96.2/10.96.{1}/"  -e "s/172.31.x.x/{2}/" cni-{1}.yaml
-
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' kubectl --kubeconfig=kubeconfig-{1} apply -f cni-{1}.yaml
-
sed -i 's!kubectl!kubectl --kubeconfig=/etc/kubernetes/admin.conf!' set-label.sh 
cat masters.txt | parallel -j1000 scp -i /tmp/aaa.pem set-label.sh centos@{}:/tmp
cat masters.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo bash /tmp/set-label.sh
-
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' kubectl --kubeconfig=kubeconfig-{1} create -f first-containers.yaml
```

## Nested kubernetes installation on openstack

Nested kubernetes installation can be tried on all-in-one openstack node.

After that node is intalled by ansible-deployer,
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#openstack-1

one subnet for virtual-network will be created
 - for example, underlay1, 10.0.1.0/24, SNAT is checked

Additionally, link local service for vRouter TCP/9091 need to be manually created
 - https://github.com/Juniper/contrail-kubernetes-docs/blob/master/install/kubernetes/nested-kubernetes.md#option-1-fabric-snat--link-local-preferred

This configuration will create DNAT/SNAT, such as from src: 10.0.1.3:xxxx, dst-ip: 10.1.1.11:9091 to src: compute's vhost0 ip:xxxx dst-ip: 127.0.0.1:9091, so CNI inside openstack VM can directly communicate to vrouter-agent on the compute node, and pick port / ip info for containers.
 - ip address can be either one from the subnet or outside of the subnet.

On that node, two Centos7 (or ubuntu bionic) nodes will be created, and kubernetes cluster will be installed with the same procedure with this url,
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#kubeadm

although yaml file needs to the one for nested install.
```
./resolve-manifest.sh contrail-nested-kubernetes.yaml > cni-tungsten-fabric.yaml

KUBEMANAGER_NESTED_MODE: "{{ KUBEMANAGER_NESTED_MODE }}" ## this needs to be "1"
KUBERNESTES_NESTED_VROUTER_VIP: {{ KUBERNESTES_NESTED_VROUTER_VIP }} ## this parameter needs to be the same IP with the one defined in link-local service (such as 10.1.1.11)
```

If coredns received ip, nested installation is working fine.

## Tungsten fabric deployment on public cloud

### gatewayless and snat

When installed on public cloud, vRouter needs to have floating-ip from underlay IP, since no hardware which serve MPLS over IP or VXLAN is avalaible ..

Having said that, since tungsten fabric supports gatewayless feature, it won't have much difficulty to serve floating-ip from that virtual-network (idea is to attach another IP with ENI, and make it the source of floating-ip, to make the service on vRouter accessible from the outside world)
 - https://github.com/Juniper/contrail-specs/blob/master/gateway-less-forwarding.md

Note: I personally prefer to make service-network gatewayless, when kubernetes is used (external-ip will not be used for this setup). If some hypervisor with baremetal instance is used, floating-ip with some gatewayless subnet is still the pereferred option.

From vRouter to the outside world, Distribute SNAT feature would do the trick.
 - https://github.com/Juniper/contrail-specs/blob/master/distributed-snat.md
 - SNAT with floating-ip also would work well

### AZ high availability WIP

Additionally, it would be also possible to define two separate load balancers on vRouters to reach the same application, to make it accessble from two different avaibility zones, which potentially ensure higher availablity.

To set up this, several things need to be configured.
 1. vRouter has one gatewayless subnet whose subnet range is not included in VPC subnet.
 2. one instance route will be configured in route table, which will forward gatewayless subnet to one of vRouter node.
 3. ELB will be set up with IP address of gatewayless IP. (When ip is specified, 'Other IP address' need to be configured for 'Subnet')
 4. Since in this case, security-group didn't automatically allow ELB's address, VPC's CIDR need to be manually allowed for health check from ELB to work well.

One limitation is vRouter's gatewayless can forward packets to other vRouter only when destination vRouter is in the same L2 subnet with the vRouter which originally received the packet.
 - if that is in the different subnet, vRouter will forward that packet to VROUTER_GATEWAY, which in turn, send that packet to vRouter to form a loop ..

Since AWS subnet cannot contain same subnet, to make this setup AZ HA, two load-balancers to the same application need to be configured, with two different gatewayless subnet for each AZ.

Since ELB can forward packet to two vRouter load-balancers, it can be AZ HA with the help of ELB.

### EKS integration

vRouter CNI AWS EKS is another possible integration scenario.
 - https://www.youtube.com/channel/UCXUny7HKBdyakn3-UOdkhsw

To setup this, firstly EKS will be configured with some IAM user from web console.
 - Since kubectl can be used with that IAM user which create that kubernetes cluster, root access for web console is not recommended.

Then, these commands remove VPC CNI from each worker node.
```
(laptop)
# kubectl delete ds -n kube-system aws-node
(EKS worker node)
# mv -i /etc/cni/net.d/10-aws.conflist /tmp/
```

After that, the same procedure with this URL can be used to install vRouter CNI.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#kubeadm

Note: 
Although by default vrouter.ko from dockerhub container cannot be loaded in amazon linux 2 kernel, this procedure can  be used to create vrouter.ko for that kernel.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricKnowledgeBase.md#how-to-build-tungsten-fabric

### CNI MTU setting

One note, when vRouter is installed in public cloud instances, some MTU issue might be seen.

Most of the issue will be fixed when physical interface MTU is changed, but when packets from containers is fragmented, CNI's MTU setting might need to be changed.

```
vi /etc/cni/net.d/10-contrail.conf
{
    "cniVersion": "0.3.1",
    "contrail" : {
        "meta-plugin"   : "$KUBERNETES_CNI_META_PLUGIN",
        "vrouter-ip"    : "127.0.0.1",
        "vrouter-port"  : $VROUTER_PORT,
        "config-dir"    : "/var/lib/contrail/ports/vm",
        "poll-timeout"  : 5,
        "poll-retries"  : 15,
+       "mtu"  : 1300,
        "log-file"      : "$LOG_DIR/cni/opencontrail.log",
        "log-level"     : "4"
    },
    "name": "contrail-k8s-cni",
    "type": "contrail-k8s-cni"
}
```

https://github.com/Juniper/contrail-container-builder/blob/master/containers/kubernetes/cni-init/entrypoint.sh#L34
https://github.com/Juniper/contrail-controller/blob/master/src/container/cni/contrail/cni.go#L33


## erm-vpn
When erm-vpn is enabled, vrouter send multicast traffic to up to 4 nodes, to avoid ingress replication to all the nodes.
Control implements a tree to send multicast packets to all nodes.
 - https://tools.ietf.org/html/draft-marques-l3vpn-mcast-edge-00
 - https://review.opencontrail.org/c/Juniper/contrail-controller/+/256

To illustrate this feature, I created a cluster with 20 kubernetes workers, and deployment with 20 replica.
 - default-k8s-pod-network is not used, since it is l3-only forwarding mode. Instead, vn1 (10.0.1.0/24) is manually defined

In this setup, this command will dump the next-hops which will send overlay BUM traffic.
 - vrf 2 has vn1's route on each worker node
 - all.txt has 20 nodes' IP
 - When BUM packets are sent from containers on 'Introspect Host', they are sent to 'dip' address by unicast overlay

```
[root@ip-172-31-12-135 ~]# for i in $(cat all.txt); do ./contrail-introspect-cli/ist.py --host $i vr route -v 2 --family layer2 ff:ff:ff:ff:ff:ff -r | grep -w -e dip -e Introspect | sort -r | uniq ; done
Introspect Host: 172.31.15.27
                  dip: 172.31.7.18
Introspect Host: 172.31.4.249
                  dip: 172.31.9.151
                  dip: 172.31.9.108
                  dip: 172.31.8.233
                  dip: 172.31.2.127
                  dip: 172.31.10.233
Introspect Host: 172.31.14.220
                  dip: 172.31.7.6
Introspect Host: 172.31.8.219
                  dip: 172.31.3.56
Introspect Host: 172.31.7.223
                  dip: 172.31.3.56
Introspect Host: 172.31.2.127
                  dip: 172.31.7.6
                  dip: 172.31.7.18
                  dip: 172.31.4.249
                  dip: 172.31.3.56
Introspect Host: 172.31.14.255
                  dip: 172.31.7.6
Introspect Host: 172.31.7.6
                  dip: 172.31.2.127
                  dip: 172.31.14.255
                  dip: 172.31.14.220
                  dip: 172.31.13.115
                  dip: 172.31.11.208
Introspect Host: 172.31.10.233
                  dip: 172.31.4.249
Introspect Host: 172.31.15.232
                  dip: 172.31.7.18
Introspect Host: 172.31.9.108
                  dip: 172.31.4.249
Introspect Host: 172.31.8.233
                  dip: 172.31.4.249
Introspect Host: 172.31.8.206
                  dip: 172.31.3.56
Introspect Host: 172.31.7.142
                  dip: 172.31.3.56
Introspect Host: 172.31.15.210
                  dip: 172.31.7.18
Introspect Host: 172.31.11.208
                  dip: 172.31.7.6
Introspect Host: 172.31.13.115
                  dip: 172.31.9.151
Introspect Host: 172.31.7.18
                  dip: 172.31.2.127
                  dip: 172.31.15.27
                  dip: 172.31.15.232
                  dip: 172.31.15.210
Introspect Host: 172.31.3.56
                  dip: 172.31.8.219
                  dip: 172.31.8.206
                  dip: 172.31.7.223
                  dip: 172.31.7.142
                  dip: 172.31.2.127
Introspect Host: 172.31.9.151
                  dip: 172.31.13.115
[root@ip-172-31-12-135 ~]# 
```

As an example, I tried to send a ping to multicast address ($ ping 224.0.0.1) from a container on worker 172.31.7.18, and it sent 4 packets to compute nodes which is included in dip list.

```
[root@ip-172-31-7-18 ~]# tcpdump -nn -i eth0 -v udp port 6635

15:02:29.883608 IP (tos 0x0, ttl 64, id 0, offset 0, flags [none], proto UDP (17), length 170)
    172.31.7.18.63685 > 172.31.2.127.6635: UDP, length 142
15:02:29.883623 IP (tos 0x0, ttl 64, id 0, offset 0, flags [none], proto UDP (17), length 170)
    172.31.7.18.63685 > 172.31.15.27.6635: UDP, length 142
15:02:29.883626 IP (tos 0x0, ttl 64, id 0, offset 0, flags [none], proto UDP (17), length 170)
    172.31.7.18.63685 > 172.31.15.210.6635: UDP, length 142
15:02:29.883629 IP (tos 0x0, ttl 64, id 0, offset 0, flags [none], proto UDP (17), length 170)
    172.31.7.18.63685 > 172.31.15.232.6635: UDP, length 142
```

And other nodes (172.31.7.223), which is not defined as direct next-hop, also received multicast packet, although it has some increased latency.
 - In this case, 2 extra hop is needed: 172.31.7.18 -> 172.31.2.127 -> 172.31.3.56 ->172.31.7.223 

```
[root@ip-172-31-7-223 ~]# tcpdump -nn -i eth0 -v udp port 6635

15:02:29.884070 IP (tos 0x0, ttl 64, id 0, offset 0, flags [none], proto UDP (17), length 170)
    172.31.3.56.56541 > 172.31.7.223.6635: UDP, length 142
```


## vRouter ml2 plugin 

I tried ml2 feature of vRouter neutron plugin.
 - https://opendev.org/x/networking-opencontrail/
 - https://www.youtube.com/watch?v=4MkkMRR9U2s

Three CentOS7.5 (4 cpu, 16 GB mem, 30 GB disk, ami: ami-3185744e) on aws are used.

Procedure based on this document is attached.
 - https://opendev.org/x/networking-opencontrail/src/branch/master/doc/source/installation/playbooks.rst

```
openstack-controller: 172.31.15.248
tungsten-fabric-controller (vRouter): 172.31.10.212
nova-compute (ovs): 172.31.0.231

(commands are on tungsten-fabric-controller, centos user is used (not root))

sudo yum -y remove PyYAML python-requests
sudo yum -y install git patch
sudo easy_install pip
sudo pip install PyYAML requests ansible==2.8.8
ssh-keygen
 add id_rsa.pub to authorized_keys on all three nodes (centos user (not root))

git clone https://opendev.org/x/networking-opencontrail.git
cd networking-opencontrail
patch -p1 < ml2-vrouter.diff 

cd playbooks
cp -i hosts.example hosts
cp -i group_vars/all.yml.example group_vars/all.yml

(ssh to all the nodes once, to update known_hosts)

ansible-playbook main.yml -i hosts

 - devstack's logs are in /opt/stack/logs/stack.sh.log
 - openstack processes' log are written in /var/log/messages
 - 'systemctl list-unit-files | grep devstack' show systemctl entry of openstack processes

(openstack controller node)
 if devstack failed with mariadb login error, please type this to fix. (last two lines need to be modified for openstack controller's ip and fqdn)
 commands will be typed by 'centos' user (not root).
   mysqladmin -u root password admin
   mysql -uroot -padmin -h127.0.0.1 -e 'GRANT ALL PRIVILEGES ON *.* TO '\''root'\''@'\''%'\'' identified by '\''admin'\'';'
   mysql -uroot -padmin -h127.0.0.1 -e 'GRANT ALL PRIVILEGES ON *.* TO '\''root'\''@'\''172.31.15.248'\'' identified by '\''admin'\'';'
   mysql -uroot -padmin -h127.0.0.1 -e 'GRANT ALL PRIVILEGES ON *.* TO '\''root'\''@'\''ip-172-31-15-248.ap-northeast-1.compute.internal'\'' identified by '\''admin'\'';'
```

hosts, group_vars/all, and patch is attached. (some are bugfix only, but some change the default behavior)
```
[centos@ip-172-31-10-212 playbooks]$ cat hosts
controller ansible_host=172.31.15.248 ansible_user=centos

# This host should be one from the compute host group.
# Playbooks are not prepared to deploy tungsten fabric compute node separately.
contrail_controller ansible_host=172.31.10.212 ansible_user=centos local_ip=172.31.10.212

[contrail]
contrail_controller

[openvswitch]
other_compute ansible_host=172.31.0.231 local_ip=172.31.0.231 ansible_user=centos

[compute:children]
contrail
openvswitch
[centos@ip-172-31-10-212 playbooks]$ cat group_vars/all.yml
---
# IP address for OpenConrail (e.g. 192.168.0.2)
contrail_ip: 172.31.10.212

# Gateway address for OpenConrail (e.g. 192.168.0.1)
contrail_gateway:

# Interface name for OpenConrail (e.g. eth0)
contrail_interface:

# IP address for OpenStack VM (e.g. 192.168.0.3)
openstack_ip: 172.31.15.248

# OpenStack branch used on VMs.
openstack_branch: stable/queens

# Optionally use different plugin version (defaults to value of OpenStack branch)
networking_plugin_version: master

# Tungsten Fabric docker image tag for contrail-ansible-deployer
contrail_version: master-latest

# If true, then install networking_bgpvpn plugin with contrail driver
install_networking_bgpvpn_plugin: false

# If true, integration with Device Manager will be enabled and vRouter
# encapsulation priorities will be set to 'VXLAN,MPLSoUDP,MPLSoGRE'.
dm_integration_enabled: false

# Optional path to file with topology for DM integration. When set and
# DM integration enabled, topology.yaml file will be copied to this location
dm_topology_file:

# If true, password to the created instances for current ansible user
# will be set to value of instance_password
change_password: false
# instance_password: uberpass1

# If set, override docker deamon /etc config file with this data
# docker_config:
[centos@ip-172-31-10-212 playbooks]$ 



[centos@ip-172-31-10-212 networking-opencontrail]$ cat ml2-vrouter.diff 
diff --git a/playbooks/roles/contrail_node/tasks/main.yml b/playbooks/roles/contrail_node/tasks/main.yml
index ee29b05..272ee47 100644
--- a/playbooks/roles/contrail_node/tasks/main.yml
+++ b/playbooks/roles/contrail_node/tasks/main.yml
@@ -7,7 +7,6 @@
       - epel-release
       - gcc
       - git
-      - ansible-2.4.*
       - yum-utils
       - libffi-devel
     state: present
@@ -61,20 +60,20 @@
     chdir: ~/contrail-ansible-deployer/
     executable: /bin/bash
 
-- name: Generate ssh key for provisioning other nodes
-  openssh_keypair:
-    path: ~/.ssh/id_rsa
-    state: present
-  register: contrail_deployer_ssh_key
-
-- name: Propagate generated key
-  authorized_key:
-    user: "{{ ansible_user }}"
-    state: present
-    key: "{{ contrail_deployer_ssh_key.public_key }}"
-  delegate_to: "{{ item }}"
-  with_items: "{{ groups.contrail }}"
-  when: contrail_deployer_ssh_key.public_key
+#- name: Generate ssh key for provisioning other nodes
+#  openssh_keypair:
+#    path: ~/.ssh/id_rsa
+#    state: present
+#  register: contrail_deployer_ssh_key
+#
+#- name: Propagate generated key
+#  authorized_key:
+#    user: "{{ ansible_user }}"
+#    state: present
+#    key: "{{ contrail_deployer_ssh_key.public_key }}"
+#  delegate_to: "{{ item }}"
+#  with_items: "{{ groups.contrail }}"
+#  when: contrail_deployer_ssh_key.public_key
 
 - name: Provision Node before deploy contrail
   shell: |
@@ -105,4 +104,4 @@
     sleep: 5
     host: "{{ contrail_ip }}"
     port: 8082
-    timeout: 300
\ No newline at end of file
+    timeout: 300
diff --git a/playbooks/roles/contrail_node/templates/instances.yaml.j2 b/playbooks/roles/contrail_node/templates/instances.yaml.j2
index e3617fd..81ea101 100644
--- a/playbooks/roles/contrail_node/templates/instances.yaml.j2
+++ b/playbooks/roles/contrail_node/templates/instances.yaml.j2
@@ -14,6 +14,7 @@ instances:
       config_database:
       config:
       control:
+      analytics:
       webui:
 {% if "contrail_controller" in groups["contrail"] %}
       vrouter:
diff --git a/playbooks/roles/docker/tasks/main.yml b/playbooks/roles/docker/tasks/main.yml
index 8d7971b..5ed9352 100644
--- a/playbooks/roles/docker/tasks/main.yml
+++ b/playbooks/roles/docker/tasks/main.yml
@@ -6,7 +6,6 @@
       - epel-release
       - gcc
       - git
-      - ansible-2.4.*
       - yum-utils
       - libffi-devel
     state: present
@@ -62,4 +61,4 @@
       - docker-py==1.10.6
       - docker-compose==1.9.0
     state: present
-    extra_args: --user
\ No newline at end of file
+    extra_args: --user
diff --git a/playbooks/roles/node/tasks/main.yml b/playbooks/roles/node/tasks/main.yml
index 0fb1751..d9ab111 100644
--- a/playbooks/roles/node/tasks/main.yml
+++ b/playbooks/roles/node/tasks/main.yml
@@ -1,13 +1,21 @@
 ---
-- name: Update kernel
+- name: Install required utilities
   become: yes
   yum:
-    name: kernel
-    state: latest
-  register: update_kernel
+    name:
+      - python3-devel
+      - libibverbs  ## needed by openstack controller node
+    state: present
 
-- name: Reboot the machine
-  become: yes
-  reboot:
-  when: update_kernel.changed
-  register: reboot_machine
+#- name: Update kernel
+#  become: yes
+#  yum:
+#    name: kernel
+#    state: latest
+#  register: update_kernel
+#
+#- name: Reboot the machine
+#  become: yes
+#  reboot:
+#  when: update_kernel.changed
+#  register: reboot_machine
diff --git a/playbooks/roles/restack_node/tasks/main.yml b/playbooks/roles/restack_node/tasks/main.yml
index a11e06e..f66d2ee 100644
--- a/playbooks/roles/restack_node/tasks/main.yml
+++ b/playbooks/roles/restack_node/tasks/main.yml
@@ -9,7 +9,7 @@
   become: yes
   pip:
     name:
-      - setuptools
+      - setuptools==43.0.0
       - requests
     state: forcereinstall
 

[centos@ip-172-31-10-212 networking-opencontrail]$ 

```



About 50 minutes is needed for installation to finish. 

Although /home/centos/devstack/openrc can be used for 'demo' user login, admin access is needed to specify network-type (empty for vRouter, 'vxlan' for ovs),
so adminrc need to be manually created.


```
[centos@ip-172-31-15-248 ~]$ cat adminrc 
export OS_PROJECT_DOMAIN_ID=default
export OS_REGION_NAME=RegionOne
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_IDENTITY_API_VERSION=3
export OS_PASSWORD=admin
export OS_AUTH_TYPE=password
export OS_AUTH_URL=http://172.31.15.248/identity  ## this needs to be modified
export OS_USERNAME=admin
export OS_TENANT_NAME=admin
export OS_VOLUME_API_VERSION=2
[centos@ip-172-31-15-248 ~]$ 


openstack network create testvn
openstack subnet create --subnet-range 192.168.100.0/24 --network testvn subnet1
openstack network create --provider-network-type vxlan testvn-ovs
openstack subnet create --subnet-range 192.168.110.0/24 --network testvn-ovs subnet1-ovs

 - two virtual-networks are created
[centos@ip-172-31-15-248 ~]$ openstack network list
+--------------------------------------+------------+--------------------------------------+
| ID                                   | Name       | Subnets                              |
+--------------------------------------+------------+--------------------------------------+
| d4e08516-71fc-401b-94fb-f52271c28dc9 | testvn-ovs | 991417ab-7da5-44ed-b686-8a14abbe46bb |
| e872b73e-100e-4ab0-9c53-770e129227e8 | testvn     | 27d828eb-ada4-4113-a6f8-df7dde2c43a4 |
+--------------------------------------+------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$

 - testvn's provider:network_type is empty

[centos@ip-172-31-15-248 ~]$ openstack network show testvn
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2020-01-18T16:14:42Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e872b73e-100e-4ab0-9c53-770e129227e8 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | testvn                               |
| port_security_enabled     | True                                 |
| project_id                | 84a573dbfadb4a198ec988e36c4f66f6     |
| provider:network_type     | local                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 27d828eb-ada4-4113-a6f8-df7dde2c43a4 |
| tags                      |                                      |
| updated_at                | 2020-01-18T16:14:44Z                 |
+---------------------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$ 

 - and it is created in tungsten fabric's db

(venv) [root@ip-172-31-10-212 ~]# contrail-api-cli --host 172.31.10.212 ls -l virtual-network
virtual-network/e872b73e-100e-4ab0-9c53-770e129227e8  default-domain:admin:testvn
virtual-network/5a88a460-b049-4114-a3ef-d7939853cb13  default-domain:default-project:dci-network
virtual-network/f61d52b0-6577-42e0-a61f-7f1834a2f45e  default-domain:default-project:__link_local__
virtual-network/46b5d74a-24d3-47dd-bc82-c18f6bc706d7  default-domain:default-project:default-virtual-network
virtual-network/52925e2d-8c5d-4573-9317-2c346fb9edf0  default-domain:default-project:ip-fabric
virtual-network/2b0469cf-921f-4369-93a7-2d73350c82e7  default-domain:default-project:_internal_vn_ipv6_link_local
(venv) [root@ip-172-31-10-212 ~]# 


 - on the other hand, testvn-ovs's provider:network_type is vxlan, and segmentation id, mtu is automatically specified

[centos@ip-172-31-15-248 ~]$ openstack network show testvn
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2020-01-18T16:14:42Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e872b73e-100e-4ab0-9c53-770e129227e8 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | testvn                               |
| port_security_enabled     | True                                 |
| project_id                | 84a573dbfadb4a198ec988e36c4f66f6     |
| provider:network_type     | local                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 27d828eb-ada4-4113-a6f8-df7dde2c43a4 |
| tags                      |                                      |
| updated_at                | 2020-01-18T16:14:44Z                 |
+---------------------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$ openstack network show testvn-ovs
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2020-01-18T16:14:47Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | d4e08516-71fc-401b-94fb-f52271c28dc9 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | testvn-ovs                           |
| port_security_enabled     | True                                 |
| project_id                | 84a573dbfadb4a198ec988e36c4f66f6     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 50                                   |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 991417ab-7da5-44ed-b686-8a14abbe46bb |
| tags                      |                                      |
| updated_at                | 2020-01-18T16:14:49Z                 |
+---------------------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$ 

```

## CentOS 8 installation procedure

```
centos8.2
ansible-deployer is used

only python3 is used (no python2)
 - ansible 2.8.x is required

1 x for tf-controller and kube-master, 1 vRouter

(all nodes)
yum install python3 chrony
alternatives --set python /usr/bin/python3

(vRouter nodes)
yum install network-scripts
 - this is required, since vRouter currently doesn't support NetworkManager

(ansible node)
sudo yum -y install git
sudo pip3 install PyYAML requests ansible\<2.9

tungtenfabric, latest is used as container tag (if vrouter.ko didn't work, please use rhel8.2 tag for vrouter-kernel-init container, https://hub.docker.com/layers/tungstenfabric/contrail-vrouter-kernel-init/rhel8.2/images/sha256-d5744b608fe28e4ceedc3668e26827ef4008309c38d3026a85eebc9a4bd0e078)

provider_config:
  bms:
   ssh_user: root
   ssh_public_key: /root/.ssh/id_rsa.pub
   ssh_private_key: /root/.ssh/id_rsa
   domainsuffix: local
instances:
  bms1:
   provider: bms
   roles:
      config_database:
      config:
      control:
      analytics:
      analytics_database:
      webui:
      k8s_master:
      kubemanager:
   ip: 172.31.14.47 ## k8s master's ip
  bms2:
   provider: bms
   roles:
     vrouter:
     k8s_node:
   ip: 172.31.41.236 ## k8s node's ip
contrail_configuration:
  CONTRAIL_CONTAINER_TAG: latest
  KUBERNETES_CLUSTER_PROJECT: {}
  JVM_EXTRA_OPTS: "-Xms128m -Xmx1g"
global_configuration:
  CONTAINER_REGISTRY: tungstenfabric


ansible-playbook -e orchestrator=kubernetes -i inventory/ playbooks/configure_instances.yml
 - it takes about 10 minutes
ansible-playbook -e orchestrator=kubernetes -i inventory/ playbooks/install_k8s.yml
 - it takes about 5 minutes
ansible-playbook -e orchestrator=kubernetes -i inventory/ playbooks/install_contrail.yml
 - it takes about 20 minutes


- setup chrony.conf manually to make this output has * character
 chronyc -n sources


[root@ip-172-31-7-20 ~]# contrail-status 
Pod              Service         Original Name                          Original Version  State    Id            Status         
                 redis           contrail-external-redis                nightly-master    running  24f53ceab904  Up 14 minutes  
analytics        api             contrail-analytics-api                 nightly-master    running  fc149ef39805  Up 8 minutes   
analytics        collector       contrail-analytics-collector           nightly-master    running  8efa1cb83e0a  Up 8 minutes   
analytics        nodemgr         contrail-nodemgr                       nightly-master    running  28c9c62fa104  Up 8 minutes   
analytics        provisioner     contrail-provisioner                   nightly-master    running  219bd1c2dbdb  Up 8 minutes   
config           api             contrail-controller-config-api         nightly-master    running  f633fdf47eab  Up 11 minutes  
config           device-manager  contrail-controller-config-devicemgr   nightly-master    running  badce06585d9  Up 11 minutes  
config           dnsmasq         contrail-controller-config-dnsmasq     nightly-master    running  415f7264efc4  Up 11 minutes  
config           nodemgr         contrail-nodemgr                       nightly-master    running  1254133b41f8  Up 11 minutes  
config           provisioner     contrail-provisioner                   nightly-master    running  d9025a573c26  Up 11 minutes  
config           schema          contrail-controller-config-schema      nightly-master    running  56833e8b4418  Up 11 minutes  
config           stats           contrail-controller-config-stats       nightly-master    running  4409c7ca762d  Up 11 minutes  
config           svc-monitor     contrail-controller-config-svcmonitor  nightly-master    running  c2e7fb135a14  Up 11 minutes  
config-database  cassandra       contrail-external-cassandra            nightly-master    running  5c9900370ff4  Up 12 minutes  
config-database  nodemgr         contrail-nodemgr                       nightly-master    running  b4fba3a8d274  Up 12 minutes  
config-database  provisioner     contrail-provisioner                   nightly-master    running  ab513e8dae95  Up 11 minutes  
config-database  rabbitmq        contrail-external-rabbitmq             nightly-master    running  f5f4db002f59  Up 12 minutes  
config-database  zookeeper       contrail-external-zookeeper            nightly-master    running  52b0a406de26  Up 12 minutes  
control          control         contrail-controller-control-control    nightly-master    running  4b127a720fd0  Up 9 minutes   
control          dns             contrail-controller-control-dns        nightly-master    running  3853b49cc334  Up 9 minutes   
control          named           contrail-controller-control-named      nightly-master    running  cd27dca70232  Up 9 minutes   
control          nodemgr         contrail-nodemgr                       nightly-master    running  a9b4c63a5579  Up 9 minutes   
control          provisioner     contrail-provisioner                   nightly-master    running  889bfb48f1cb  Up 9 minutes   
database         cassandra       contrail-external-cassandra            nightly-master    running  077bf86be714  Up 8 minutes   
database         nodemgr         contrail-nodemgr                       nightly-master    running  e767cc9e75df  Up 8 minutes   
database         provisioner     contrail-provisioner                   nightly-master    running  f8dc9993b6f3  Up 8 minutes   
database         query-engine    contrail-analytics-query-engine        nightly-master    running  facbb4a6a7f2  Up 8 minutes   
kubernetes       kube-manager    contrail-kubernetes-kube-manager       nightly-master    running  90b2d33be774  Up 7 minutes   
webui            job             contrail-controller-webui-job          nightly-master    running  8db0224db56e  Up 10 minutes  
webui            web             contrail-controller-webui-web          nightly-master    running  00b680cdfb51  Up 10 minutes  

== Contrail control ==
control: active
nodemgr: active
named: active
dns: active

== Contrail config-database ==
nodemgr: initializing (Disk for DB is too low. )
zookeeper: active
rabbitmq: active
cassandra: active

== Contrail kubernetes ==
kube-manager: active

== Contrail database ==
nodemgr: initializing (Disk for DB is too low. )
query-engine: active
cassandra: active

== Contrail analytics ==
nodemgr: active
api: active
collector: active

== Contrail webui ==
web: active
job: active

== Contrail config ==
svc-monitor: active
nodemgr: active
device-manager: active
api: active
schema: active

[root@ip-172-31-7-20 ~]# 



[root@ip-172-31-2-120 ~]# contrail-status 
Pod      Service      Original Name           Original Version  State    Id            Status         
         rsyslogd                             nightly-master    running  74d005445632  Up 24 minutes  
vrouter  agent        contrail-vrouter-agent  nightly-master    running  d72ddaa41dd9  Up 35 seconds  
vrouter  nodemgr      contrail-nodemgr        nightly-master    running  48cf35435b13  Up 34 seconds  
vrouter  provisioner  contrail-provisioner    nightly-master    running  2a130d1531e1  Up 34 seconds  

WARNING: container with original name '' have Pod or Service empty. Pod: '' / Service: 'rsyslogd'. Please pass NODE_TYPE with pod name to container's env

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: initializing (NTP state unsynchronized. )
agent: active

[root@ip-172-31-2-120 ~]# 

[root@ip-172-31-2-120 ~]# python ist.py vr intf
+-------+----------------+--------+-------------------+---------------+---------------+---------+--------------------------------------+
| index | name           | active | mac_addr          | ip_addr       | mdata_ip_addr | vm_name | vn_name                              |
+-------+----------------+--------+-------------------+---------------+---------------+---------+--------------------------------------+
| 1     | crypt0         | Active | n/a               | n/a           | n/a           | n/a     | n/a                                  |
| 0     | eth0           | Active | n/a               | n/a           | n/a           | n/a     | n/a                                  |
| 2     | vhost0         | Active | 06:60:e9:78:16:f8 | 172.31.2.120  | 169.254.0.2   | n/a     | default-domain:default-project:ip-   |
|       |                |        |                   |               |               |         | fabric                               |
| 4     | tapeth0-87f2ec | Active | 02:c4:b6:7a:10:b9 | 10.47.255.252 | 169.254.0.4   | n/a     | default-                             |
|       |                |        |                   |               |               |         | domain:k8s-default:k8s-default-pod-  |
|       |                |        |                   |               |               |         | network                              |
| 5     | tapeth0-87f1ee | Active | 02:c5:83:a1:de:b9 | 10.47.255.251 | 169.254.0.5   | n/a     | default-                             |
|       |                |        |                   |               |               |         | domain:k8s-default:k8s-default-pod-  |
|       |                |        |                   |               |               |         | network                              |
| 3     | pkt0           | Active | n/a               | n/a           | n/a           | n/a     | n/a                                  |
+-------+----------------+--------+-------------------+---------------+---------------+---------+--------------------------------------+
[root@ip-172-31-2-120 ~]# python ist.py vr vrf
+--------------------------------------+---------+---------+---------+-----------+----------+--------------------------------------+
| name                                 | ucindex | mcindex | brindex | evpnindex | vxlan_id | vn                                   |
+--------------------------------------+---------+---------+---------+-----------+----------+--------------------------------------+
| default-domain:default-project:ip-   | 0       | 0       | 0       | 0         | 0        | N/A                                  |
| fabric:__default__                   |         |         |         |           |          |                                      |
| default-domain:default-project:ip-   | 1       | 1       | 1       | 1         | 2        | default-domain:default-project:ip-   |
| fabric:ip-fabric                     |         |         |         |           |          | fabric                               |
| default-                             | 2       | 2       | 2       | 2         | 7        | default-                             |
| domain:k8s-default:k8s-default-pod-  |         |         |         |           |          | domain:k8s-default:k8s-default-pod-  |
| network:k8s-default-pod-network      |         |         |         |           |          | network                              |
| default-                             | 3       | 3       | 3       | 3         | 8        | default-                             |
| domain:k8s-default:k8s-default-      |         |         |         |           |          | domain:k8s-default:k8s-default-      |
| service-network:k8s-default-service- |         |         |         |           |          | service-network                      |
| network                              |         |         |         |           |          |                                      |
+--------------------------------------+---------+---------+---------+-----------+----------+--------------------------------------+
[root@ip-172-31-2-120 ~]# 

[root@ip-172-31-7-20 ~]# kubectl get pod -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP              NODE                                              NOMINATED NODE   READINESS GATES
cirros-deployment-86885fbf85-7z78k   1/1     Running   0          13s   10.47.255.250   ip-172-31-2-120.ap-northeast-1.compute.internal   <none>           <none>
cirros-deployment-86885fbf85-tjkwn   1/1     Running   0          13s   10.47.255.249   ip-172-31-2-120.ap-northeast-1.compute.internal   <none>           <none>
[root@ip-172-31-7-20 ~]# 
[root@ip-172-31-7-20 ~]# 
[root@ip-172-31-7-20 ~]# kubectl exec -it cirros-deployment-86885fbf85-7z78k sh
/ # ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
17: eth0    inet 10.47.255.250/12 scope global eth0\       valid_lft forever preferred_lft forever
/ # ping 10.47.255.249
PING 10.47.255.249 (10.47.255.249): 56 data bytes
64 bytes from 10.47.255.249: seq=0 ttl=63 time=0.657 ms
64 bytes from 10.47.255.249: seq=1 ttl=63 time=0.073 ms
^C
--- 10.47.255.249 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.073/0.365/0.657 ms
/ # 


 - to make chrony works fine after vRouter installtion, chrony server might need to be restarted

[root@ip-172-31-4-206 ~]#  chronyc -n sources
210 Number of sources = 5
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^? 169.254.169.123               3   4     0   906  -8687ns[  -12us] +/-  428us
^? 129.250.35.250                2   7     0  1002   +429us[ +428us] +/-   73ms
^? 167.179.96.146                2   7     0   937   +665us[ +662us] +/- 2859us
^? 194.0.5.123                   2   6     0  1129   +477us[ +473us] +/-   44ms
^? 103.202.216.35                3   6     0   933  +9662ns[+6618ns] +/-  145ms
[root@ip-172-31-4-206 ~]# 
[root@ip-172-31-4-206 ~]# 
[root@ip-172-31-4-206 ~]# service chronyd status
Redirecting to /bin/systemctl status chronyd.service
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-06-28 16:00:34 UTC; 33min ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
 Main PID: 727 (chronyd)
    Tasks: 1 (limit: 49683)
   Memory: 2.1M
   CGroup: /system.slice/chronyd.service
           └─727 /usr/sbin/chronyd

Jun 28 16:00:33 localhost.localdomain chronyd[727]: Using right/UTC timezone to obtain leap second data
Jun 28 16:00:34 localhost.localdomain systemd[1]: Started NTP client/server.
Jun 28 16:00:42 localhost.localdomain chronyd[727]: Selected source 169.254.169.123
Jun 28 16:00:42 localhost.localdomain chronyd[727]: System clock TAI offset set to 37 seconds
Jun 28 16:19:33 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[727]: Source 167.179.96.146 offline
Jun 28 16:19:33 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[727]: Source 103.202.216.35 offline
Jun 28 16:19:33 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[727]: Source 129.250.35.250 offline
Jun 28 16:19:33 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[727]: Source 194.0.5.123 offline
Jun 28 16:19:33 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[727]: Source 169.254.169.123 offline
Jun 28 16:19:33 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[727]: Can't synchronise: no selectable sources
[root@ip-172-31-4-206 ~]# service chronyd restart
Redirecting to /bin/systemctl restart chronyd.service
[root@ip-172-31-4-206 ~]# 
[root@ip-172-31-4-206 ~]# 
[root@ip-172-31-4-206 ~]# service chronyd status
Redirecting to /bin/systemctl status chronyd.service
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-06-28 16:34:41 UTC; 2s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 25252 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 25247 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 25250 (chronyd)
    Tasks: 1 (limit: 49683)
   Memory: 1.0M
   CGroup: /system.slice/chronyd.service
           └─25250 /usr/sbin/chronyd

Jun 28 16:34:41 ip-172-31-4-206.ap-northeast-1.compute.internal systemd[1]: Starting NTP client/server...
Jun 28 16:34:41 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[25250]: chronyd version 3.5 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIVDROP +SCFILTER +SIGND>
Jun 28 16:34:41 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[25250]: Frequency 35.298 +/- 0.039 ppm read from /var/lib/chrony/drift
Jun 28 16:34:41 ip-172-31-4-206.ap-northeast-1.compute.internal chronyd[25250]: Using right/UTC timezone to obtain leap second data
Jun 28 16:34:41 ip-172-31-4-206.ap-northeast-1.compute.internal systemd[1]: Started NTP client/server.
[root@ip-172-31-4-206 ~]#  chronyc -n sources
210 Number of sources = 5
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* 169.254.169.123               3   4    17     4  -2369ns[  -27us] +/-  451us
^- 94.154.96.7                   2   6    17     5    +30ms[  +30ms] +/-  148ms
^- 185.51.192.34                 2   6    17     3  -2951us[-2951us] +/-  150ms
^- 188.125.64.6                  2   6    17     3  +9526us[+9526us] +/-  143ms
^- 216.218.254.202               1   6    17     5    +15ms[  +15ms] +/-   72ms
[root@ip-172-31-4-206 ~]# 


[root@ip-172-31-4-206 ~]# contrail-status 
Pod      Service      Original Name           Original Version  State    Id            Status         
         rsyslogd                             nightly-master    running  5fc76e57c156  Up 16 minutes  
vrouter  agent        contrail-vrouter-agent  nightly-master    running  bce023d8e6e0  Up 5 minutes   
vrouter  nodemgr      contrail-nodemgr        nightly-master    running  9439a304cbcf  Up 5 minutes   
vrouter  provisioner  contrail-provisioner    nightly-master    running  1531b1403e49  Up 5 minutes   

WARNING: container with original name '' have Pod or Service empty. Pod: '' / Service: 'rsyslogd'. Please pass NODE_TYPE with pod name to container's env

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: active
agent: active

[root@ip-172-31-4-206 ~]# 
```



## Random tungsten fabric patch (not tested)
#### static schedule for svc-monitor logic to choose available vRouters
```
diff --git a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py b
index f40de26..d5c2478 100644
--- a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
+++ b/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
@@ -200,3 +200,8 @@ class RandomScheduler(VRouterScheduler):
         self._vnc_lib.ref_update('virtual-router', chosen_vrouter,
             'virtual-machine', vm.uuid, None, 'ADD')
         return chosen_vrouter
+
+class StaticScheduler(VRouterScheduler):
+    """Statically assign vRouter nodes for v1 service-chain, haproxy lb, SNAT e
+    def schedule(self, si, vm):
+        return ['bms11', 'bms12']
```

#### decouple analytics from svc-monitor logic
```
diff --git a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py b/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.
index f40de26..7fd1f0a 100644
--- a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
+++ b/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
@@ -115,6 +115,8 @@ class VRouterScheduler(object):
         return response_dict
 
     def vrouters_running(self):
+        ## implement logic to see available vRouter, without checking analytics response (possible choice is xmpp status from control node)
+
         # get az host list
         az_vrs = self._get_az_vrouter_list()
```


https://review.opencontrail.org/c/Juniper/contrail-controller/+/59457


#### more scalable haproxy loadbalancer and SNAT
```
diff --git a/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py b/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py
index 5487b2b..1bee992 100644
--- a/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py
+++ b/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py
@@ -92,8 +92,8 @@ class OpencontrailLoadbalancerDriver(
 
         # set interfaces and ha
         props.set_interface_list(if_list)
-        props.set_ha_mode('active-standby')
-        scale_out = ServiceScaleOutType(max_instances=2, auto_scale=False)
+        props.set_ha_mode('active-active')
+        scale_out = ServiceScaleOutType(max_instances=10, auto_scale=False)
         props.set_scale_out(scale_out)
 
         return props
diff --git a/src/config/svc-monitor/svc_monitor/snat_agent.py b/src/config/svc-monitor/svc_monitor/snat_agent.py
index 54ea709..f5bce37 100644
--- a/src/config/svc-monitor/svc_monitor/snat_agent.py
+++ b/src/config/svc-monitor/svc_monitor/snat_agent.py
@@ -169,7 +169,7 @@ class SNATAgent(Agent):
             si_obj.fq_name = project_fq_name + [si_name]
             si_created = True
         si_prop_obj = ServiceInstanceType(
-            scale_out=ServiceScaleOutType(max_instances=2,
+            scale_out=ServiceScaleOutType(max_instances=10,
                                           auto_scale=True),
             auto_policy=False)
 
@@ -181,7 +181,7 @@ class SNATAgent(Agent):
         right_if = ServiceInstanceInterfaceType(
             virtual_network=':'.join(vn_obj.fq_name))
         si_prop_obj.set_interface_list([right_if, left_if])
-        si_prop_obj.set_ha_mode('active-standby')
+        si_prop_obj.set_ha_mode('active-active')
 
         si_obj.set_service_instance_properties(si_prop_obj)
         si_obj.set_service_template(st_obj)
```

#### three xmpp connections (to cover double failure scenario)
```
diff --git a/src/vnsw/agent/cmn/agent.h b/src/vnsw/agent/cmn/agent.h
index 3e48812..832b476 100644
--- a/src/vnsw/agent/cmn/agent.h
+++ b/src/vnsw/agent/cmn/agent.h
@@ -284,7 +284,10 @@ extern void RouterIdDepInit(Agent *agent);
 #define MULTICAST_LABEL_BLOCK_SIZE 2048
 
 #define MIN_UNICAST_LABEL_RANGE 4098
-#define MAX_XMPP_SERVERS 2
+
+/* to cover double failure case */
+#define MAX_XMPP_SERVERS 3 
+
 #define XMPP_SERVER_PORT 5269
 #define XMPP_DNS_SERVER_PORT 53
 #define METADATA_IP_ADDR ntohl(inet_addr("169.254.169.254"))
```


### static XMPP assignment
contrail-controller:

```
diff --git a/src/vnsw/agent/cmn/agent.cc b/src/vnsw/agent/cmn/agent.cc
index 607f384..71d27d8 100644
--- a/src/vnsw/agent/cmn/agent.cc
+++ b/src/vnsw/agent/cmn/agent.cc
@@ -469,7 +469,7 @@ void Agent::CopyFilteredParams() {
     if (new_chksum != controller_chksum_) {
         controller_chksum_ = new_chksum;
         controller_list_ = params_->controller_server_list();
-        std::random_shuffle(controller_list_.begin(), controller_list_.end());
+        std::random_shuffle(controller_list_.begin(), controller_list_.end()); // commented out for static XMPP assignment
     }
 
     // Dns
```

#### vlan-based EVPN T2 interop
```
diff --git a/src/bgp/evpn/evpn_route.cc b/src/bgp/evpn/evpn_route.cc
index 36412b2..a830b5c 100644
--- a/src/bgp/evpn/evpn_route.cc
+++ b/src/bgp/evpn/evpn_route.cc
@@ -487,7 +487,7 @@ void EvpnPrefix::BuildProtoPrefix(BgpProtoPrefix *proto_prefix,
                 proto_prefix->prefix.begin() + esi_offset);
         }
         size_t tag_offset = esi_offset + kEsiSize;
-        put_value(&proto_prefix->prefix[tag_offset], kTagSize, tag_);
+        put_value(&proto_prefix->prefix[tag_offset], kTagSize, 0);
         size_t mac_len_offset = tag_offset + kTagSize;
         proto_prefix->prefix[mac_len_offset] = 48;
         size_t mac_offset = mac_len_offset + 1;
```

#### 'enable_nova: no' can be configurable

(Implemented)

https://review.opencontrail.org/c/Juniper/contrail-kolla-ansible/+/58908

```
git clone -b contrail/queens https://github.com/Juniper/contrail-kolla-ansible

diff --git a/ansible/post-deploy-contrail.yml b/ansible/post-deploy-contrail.yml
index e603207..c700d88 100644
--- a/ansible/post-deploy-contrail.yml
+++ b/ansible/post-deploy-contrail.yml
@@ -63,6 +63,8 @@
       - ['baremetal-hosts', 'virtual-hosts']
     register: command_result
     failed_when: "command_result.rc == 1 and 'already exists' not in command_result.stderr"
+    when:
+      - enable_nova | bool
     run_once: yes
 
   - name: Add compute hosts to virtual-hosts Aggregate Group
```

#### security endpoint stats per tag as UVE
https://review.opencontrail.org/c/Juniper/contrail-specs/+/55761

#### kubernetes multi master setup

(Implemented)

1. https://review.opencontrail.org/c/Juniper/contrail-controller/+/55758
1. https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#k8sk8s

#### tc-flower offload

```
For someone interested,

I tried two vRouter setup and type those commands on one node to bypass vRouter datapath in favor of tc,
and found tc-flower based vxlan datapath (egress) and vRouter's vxlan datapath can communicate :)
 - ingress vxlan decap won't work well, I'm still investigating ..

vRouter0: 172.31.4.175 (container, 10.0.1.251)
vRouter1: 172.31.1.214 (container, 10.0.1.250, connected to tapeth0-038fdd)

[from specific tap to known ip address, vxlan encap could be offloaded to tc]
 - typed on vRouter1
ip link set vxlan7 up
ip link add vxlan7 type vxlan vni 7 dev ens5 dstport 0 external
tc filter add dev tapeth0-038fdd protocol ip parent ffff: \
                flower \
                  ip_proto icmp dst_ip 10.0.1.251 \
                action simple sdata "ttt" action tunnel_key set \
                  src_ip 172.31.1.214 \
                  dst_ip 172.31.4.175 \
                  id 7 \
                  dst_port 4789 \
                action mirred egress redirect dev vxlan7

[although for egress traffic vRouter1 is bypassed, it can still communicate]

[root@ip-172-31-1-214 ~]# tcpdump -nn -i ens5 udp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens5, link-type EN10MB (Ethernet), capture size 262144 bytes
04:55:41.566458 IP 172.31.1.214.57877 > 172.31.4.175.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.250 > 10.0.1.251: ICMP echo request, id 60416, seq 180, length 64
04:55:41.566620 IP 172.31.4.175.61117 > 172.31.1.214.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.251 > 10.0.1.250: ICMP echo reply, id 60416, seq 180, length 64
04:55:42.570917 IP 172.31.1.214.57877 > 172.31.4.175.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.250 > 10.0.1.251: ICMP echo request, id 60416, seq 181, length 64
04:55:42.571056 IP 172.31.4.175.61117 > 172.31.1.214.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.251 > 10.0.1.250: ICMP echo reply, id 60416, seq 181, length 64
^C
4 packets captured
5 packets received by filter
0 packets dropped by kernel
[root@ip-172-31-1-214 ~]#

/ # ping 10.0.1.251
PING 10.0.1.251 (10.0.1.251): 56 data bytes
64 bytes from 10.0.1.251: seq=0 ttl=64 time=5.183 ms
64 bytes from 10.0.1.251: seq=1 ttl=64 time=4.587 ms
^C
--- 10.0.1.251 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 4.587/4.885/5.183 ms
/ # 

[tap's RX is not incrementing since that is bypassed (TX increments, since ingress traffic still uses vRouter datapath)]

[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3393  bytes:288094 errors:0
            TX packets:3438  bytes:291340 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3393  bytes:288094 errors:0
            TX packets:3439  bytes:291438 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3394  bytes:288136 errors:0
            TX packets:3442  bytes:291676 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3394  bytes:288136 errors:0
            TX packets:3444  bytes:291872 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3394  bytes:288136 errors:0
            TX packets:3447  bytes:292166 errors:0
[root@ip-172-31-1-214 ~]#
```


```
contrail-controller

diff --git a/src/vnsw/agent/pkt/flow_mgmt.cc b/src/vnsw/agent/pkt/flow_mgmt.cc
index c888a26..a1b0189 100644
--- a/src/vnsw/agent/pkt/flow_mgmt.cc
+++ b/src/vnsw/agent/pkt/flow_mgmt.cc
@@ -511,6 +511,9 @@ void FlowMgmtManager::LogFlowUnlocked(FlowEntry *flow, const std::string &op) {
     FlowInfo trace;
     flow->FillFlowInfo(trace);
     FLOW_TRACE(Trace, op, trace);
+
+    // Add tc flower logic, based on FlowEntry *flow
+ 
 }
 
 // Extract all the FlowMgmtKey for a flow

```

#### vRouters on GCE cannot reach other nodes in the same subnet

when vRouter is installed in GCE, it can't reach nodes in the same subnet.
This patch is a temporary workaround.

```
diff --git a/containers/vrouter/agent/entrypoint.sh b/containers/vrouter/agent/entrypoint.sh
index f4f49f4..01e1349 100755
--- a/containers/vrouter/agent/entrypoint.sh
+++ b/containers/vrouter/agent/entrypoint.sh
@@ -140,7 +140,7 @@ if [ "$gcp" == "Google" ]; then
     for intf in $intfs ; do
         if [[ $phys_int_mac == "$(curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/network-interfaces/${intf}/mac)" ]]; then
             mask=$(curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/network-interfaces/${intf}/subnetmask)
-            vrouter_cidr=$vrouter_ip/$(mask2cidr $mask)
+            vrouter_cidr=$vrouter_ip/31  ### this can't be set /32, since in that setup, vrouter can't create ingress flow for some reason ..
         fi
     done
 fi
```

#### when it will be used with multus

(Implemented)

After this commit, vRouter works fine with multus-cni (it dynamically identify if it was directly called, or it was called by some meta plugin).
 - https://review.opencontrail.org/c/Juniper/contrail-controller/+/58387

```
(install kubernetes and vRouter by ansible-deployer: container tag: master-latest, ansible-deployer: master)
git clone https://github.com/intel/multus-cni.git && cd multus-cni
cat ./images/deprecated/multus-daemonset-pre-1.16.yml | kubectl apply -f -
```

Note: Since ansible-deployer install v0.3.0 CNI, bridge CNI didn't work well by default. When /opt/cni/bin/bridge (and /opt/cni/bin/static) file is replaced by v0.8.6 module, it worked fine.
 - https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz

#### multi vCenter setup

Tungsten fabric controller nodes serve as many vcenter-plugins as number of vCenters.

Since each vCenter has several ESXi under them, each vcenter-manager on vRouterVM on ESXi with specific vCenter need to be configured with that tenant name (instead of hard-coded 'vCenter' tenant)

```
contrail-vcenter-plugin:
diff --git a/src/net/juniper/contrail/vcenter/VCenterMonitor.java b/src/net/juniper/contrail/vcenter/VCenterMonitor.java
index d5c0043..294ee99 100644
--- a/src/net/juniper/contrail/vcenter/VCenterMonitor.java
+++ b/src/net/juniper/contrail/vcenter/VCenterMonitor.java
@@ -74,7 +74,7 @@ public class VCenterMonitor {
     private static String _authurl           = "http://10.84.24.54:35357/v2.0";
 
     private static String _zookeeperAddrPort  = "127.0.0.1:2181";
-    private static String _zookeeperLatchPath = "/vcenter-plugin";
+    private static String _zookeeperLatchPath = "/vcenter-plugin"; // make this configurable
     private static String _zookeeperId        = "node-vcenter-plugin";
 
     static volatile Mode mode  = Mode.VCENTER_ONLY;
diff --git a/src/net/juniper/contrail/vcenter/VncDB.java b/src/net/juniper/contrail/vcenter/VncDB.java
index 9d004b7..a831a37 100644
--- a/src/net/juniper/contrail/vcenter/VncDB.java
+++ b/src/net/juniper/contrail/vcenter/VncDB.java
@@ -61,8 +61,8 @@ public class VncDB {
     Mode mode;
 
     public static final String VNC_ROOT_DOMAIN     = "default-domain";
-    public static final String VNC_VCENTER_PROJECT = "vCenter";
-    public static final String VNC_VCENTER_IPAM    = "vCenter-ipam";
+    public static final String VNC_VCENTER_PROJECT = "vCenter"; // make this configurable
+    public static final String VNC_VCENTER_IPAM    = "vCenter-ipam"; // make this configurable
     public static final String VNC_VCENTER_DEFAULT_SG    = "default";
     public static final String VNC_VCENTER_PLUGIN  = "vcenter-plugin";
     public static final String VNC_VCENTER_TEST_PROJECT = "vCenter-test";


contrail-vcenter-manager:
diff --git a/cvm/constants.py b/cvm/constants.py
index 0dcabab..4b30299 100644
--- a/cvm/constants.py
+++ b/cvm/constants.py
@@ -31,8 +31,8 @@ VM_UPDATE_FILTERS = [
     'runtime.powerState',
 ]
 VNC_ROOT_DOMAIN = 'default-domain'
-VNC_VCENTER_PROJECT = 'vCenter'
-VNC_VCENTER_IPAM = 'vCenter-ipam'
+VNC_VCENTER_PROJECT = 'vCenter' ## make this configurable
+VNC_VCENTER_IPAM = 'vCenter-ipam' ## make this configurable
 VNC_VCENTER_IPAM_FQN = [VNC_ROOT_DOMAIN, VNC_VCENTER_PROJECT, VNC_VCENTER_IPAM]
 VNC_VCENTER_DEFAULT_SG = 'default'
 VNC_VCENTER_DEFAULT_SG_FQN = [VNC_ROOT_DOMAIN, VNC_VCENTER_PROJECT, VNC_VCENTER_DEFAULT_SG]
```

#### use same ECMP hash on all compute nodes for possible symmetric ECMP in packet-mode

(Implemented)
 - https://review.opencontrail.org/c/Juniper/contrail-controller/+/57643

https://review.opencontrail.org/c/Juniper/contrail-controller/+/32223

```
diff --git a/src/vnsw/agent/pkt/pkt_handler.cc b/src/vnsw/agent/pkt/pkt_handler.cc
index 28e5637..075bb17 100644
--- a/src/vnsw/agent/pkt/pkt_handler.cc
+++ b/src/vnsw/agent/pkt/pkt_handler.cc
@@ -1304,7 +1304,7 @@ std::size_t PktInfo::hash(const Agent *agent,
     // We need to ensure that hash computed in Compute-1 and Compute-2 are
     // different. We also want to have same hash on agent restarts. So, include
     // vhost-ip also to compute hash
-    boost::hash_combine(seed, agent->router_id().to_ulong());
+    ////// boost::hash_combine(seed, agent->router_id().to_ulong());
 
     if (family == Address::INET) {
         if (ecmp_load_balance.is_source_ip_set()) {

```

#### specify vlan-id when transparent service-chain is used

https://github.com/Juniper/contrail-controller/blob/R1907/src/config/schema-transformer/config_db.py#L3060
```
# diff -u config_db.py.orig config_db.py
--- config_db.py.orig 2019-08-04 10:54:22.993291899 +0000
+++ config_db.py 2019-08-04 13:05:23.665843100 +0000
@@ -3059,6 +3062,21 @@
                                     service_ri1, service_ri2):
         vlan = self._object_db.allocate_service_chain_vlan(vm_info['vm_uuid'],
                                                            self.name)
+        ####
+        ## vlan-id is embedded in service-instance name
+        ## servicename---vm_uuid---vlanid
+        ####
+        for servicename in self.service_list:
+          left_interface_uuid = vm_info['left']['vmi'].name.split (':')[-1]
+          if (servicename.find(left_interface_uuid ) > -1):
+            vlan = servicename.split(':')[-1].split('---')[-1]
+
         self.add_pbf_rule(vm_info['left']['vmi'], service_ri1,
                           v4_address, v6_address, vlan)
         self.add_pbf_rule(vm_info['right']['vmi'], service_ri2,
@@ -3911,6 +3929,22 @@
                 vlan = self._object_db.allocate_service_chain_vlan(
                     vm_pt.uuid, service_chain.name)

+
+                ###
+                # begin: added
+                ###
+                for servicename in service_chain.service_list:
+                  if (servicename.find(self.name.split(':')[-1]) > -1):
+                    vlan = servicename.split(':')[-1].split('---')[-1]
+                ###
+                # end: added
+                ###
+
                 service_chain.add_pbf_rule(self, service_ri, v4_address,
                                            v6_address, vlan)
             #end for service_chain
```


#### support older kernel for CentOS

Juniper/contrail-packages

````
diff --git a/kernel_version.info b/kernel_version.info
index 8d38f34..d5e711b 100644
--- a/kernel_version.info
+++ b/kernel_version.info
@@ -1,2 +1,3 @@
+3.10.0-862.2.3.el7.x86_64
 3.10.0-1062.4.1.el7.x86_64
-3.10.0-1062.9.1.el7.x86_64
\ No newline at end of file
+3.10.0-1062.9.1.el7.x86_64
````

#### configurable minimum route target id

```
diff --git a/src/config/common/cfgm_common/__init__.py b/src/config/common/cfgm_common/__init__.py
index 088b03b..dd484ab 100644
--- a/src/config/common/cfgm_common/__init__.py
+++ b/src/config/common/cfgm_common/__init__.py
@@ -18,7 +18,7 @@ DCI_VN_FQ_NAME = ['default-domain', 'default-project', 'dci-network']
 DCI_IPAM_FQ_NAME = ['default-domain', 'default-project', 'default-dci-lo0-network-ipam']
 OVERLAY_LOOPBACK_FQ_PREFIX = ['default-domain', 'default-project']
 
-_BGP_RTGT_MIN_ID_TYPE0 = 8000000
+_BGP_RTGT_MIN_ID_TYPE0 = 8100000
 _BGP_RTGT_MIN_ID_TYPE1_2 = 8000
 SGID_MIN_ALLOC = 8000000
 VNID_MIN_ALLOC = 1
```


#### vRouter build with linux 5.x kernel fail
(Implemented)
https://review.opencontrail.org/c/tungstenfabric/tf-vrouter/+/60771

// https://review.opencontrail.org/c/Juniper/contrail-vrouter/+/57506

