
Table of Contents
=================

   * [A Tungsten Fabric Primer](#a-tungsten-fabric-primer)
      * [1. Why Tungsten Fabric?](#1-why-tungsten-fabric)
         * [I. Interoperability with ASIC](#i-interoperability-with-asic)
         * [II. Scalability](#ii-scalability)
      * [2. TungstenFabric, Up and Running](#2-tungstenfabric-up-and-running)
         * [Appendix: external access](#appendix-external-access)
      * [3. After this reading](#3-after-this-reading)
   * [Components in Tungsten Fabric](#components-in-tungsten-fabric)
      * [Overall picture](#overall-picture)
      * [control, vRouter](#control-vrouter)
      * [config (config-api, schema-transformer, svc-monitor)](#config-config-api-schema-transformer-svc-monitor)
        * [schema-transformer](#schema-transformer)
        * [svc-monitor](#svc-monitor)
      * [config-database (zookeeper, cassandra, rabbitmq)](#config-database-zookeeper-cassandra-rabbitmq)
      * [nodemgr](#nodemgr)
      * [analytics](#analytics)
      * [analytics-database](#analytics-database)
      * [webui (webui-web, webui-job)](#webui-webui-web-webui-job)
   * [Orchestrator integration](#orchestrator-integration)
      * [Openstack](#openstack)
      * [kubernetes](#kubernetes)
      * [vCenter](#vcenter)
   * [More on Installation](#more-on-installation)
      * [HA behavior of Tungsten Fabric components](#ha-behavior-of-tungsten-fabric-components)
      * [Multi-NIC installation](#multi-nic-installation)
      * [kubeadm](#kubeadm)
      * [Openstack](#openstack-1)
      * [vCenter](#vcenter-1)
   * [Monitoring integration](#monitoring-integration)
      * [Prometheus](#prometheus)
      * [EFK](#efk)
   * [Day 2 operation](#day-2-operation)
      * [ist.py](#istpy)
      * [contrail-api-cli](#contrail-api-cli)
      * [webui](#webui)
   * [Appendix](#appendix)
      * [L3VPN / EVPN (T2/T5, VXLAN/MPLS) integration](#l3vpn--evpn-t2t5-vxlanmpls-integration)
      * [Service-Chain (L2, L3, NAT), BGPaaS](#service-chain-l2-l3-nat-bgpaas)
      * [Multicluster](#multicluster)
         * [routing / bridging, DNS, security-policy](#routing--bridging-dns-security-policy)
         * [Inter AS option B/C](#inter-as-option-bc)
      * [Multi orchestrator](#multi-orchestrator)
         * [k8s openstack](#k8sopenstack)
         * [k8s k8s](#k8sk8s)
         * [openstack openstack](#openstackopenstack)
         * [k8s vCenter](#k8svcenter)
         * [openstack vCenter](#openstackvcenter)
         * [vCenter vCenter](#vcentervcenter)
         * [k8s openstack vCenter](#k8sopenstackvcenter)
      * [Service Mesh](#service-mesh)


# A Tungsten Fabric Primer

Let me briefly describe what I have learnt in this two years journey around Tungsten Fabric.
https://tungsten.io/

## 1. Why Tungsten Fabric?

There are a lot of good implementation of SDN/neutron/CNI, so why try another one is an important point.
AFAIK, TungstenFabric has two key differentiators, which makes that unique one around.

### I. Interoperability with ASIC

Although there are a lot of technology which makes linux software a good candidate of production router/switch,
ASIC still is a vital part of this industry.
To interoperate with them, SDN platform need some routing protocol, such as bgp or ovsdb.

To makes things more complex, many service providers and cloud providers use VRFs to terminate and separate each customer's network connection, which makes stitching between Routers and SDNs makes a complex task.
 - Mostly, vlans can be used between them, but termination points at SDN platform could be a source of bottleneck
 - Moreover, each SDN termination points (similar to network nodes in openstack) need to have separate configuration per customer, which makes configuration more complex

TungstenFabric resolved this issue, with the help of mature implementation of MP-BGP, which allows each VRF on the routers to send packets directly to vRouters which serves each customer's application.
This feature allows horizontal scaling of compute nodes with separate networks per customer, to be based on control plane rather than data plane, and makes that much more intuitive.


### II. Scalability

Since packets are sent directly from routers to vRouters, there is no need for network nodes, which makes Tungsten Fabric much more scalable, in terms of data plane.

Moreover, in control plane perspective, it has a curious feature named route target filtering (https://tools.ietf.org/html/rfc4684).
 - This feature is common in MP-BGP and other routers also has that feature
 - This feature means if vRouters doesn't have a prefix with that route-target, control plane drop that prefix when it received

Since in cloud service, customer uses limited part of cloud providers' DCs, and different customer will use different route-target, vRouters and controllers don't need to know all the prefixes.
Route target filtering feature makes that behavior possible, and dramatically reduce the number of prefixes each vRouter (and each controller if RR is used between them) needs to take care of, which makes this control plane much more scalable.


Combininng them with other features like security-policy, network-policy/logical-router (it is similar to VPC peerling or transit-gateway in AWS), I think it will be a good candidate of VPC infrastructure (similar to AWS/Azure/GCP VPC/vnet) for both of private cloud or managed cloud world, and that makes it so interesting platform which is worth a try.


## 2. TungstenFabric, Up and Running

To try TungstenFabric for the first time, I recommend using ansible-deployer (https://github.com/Juniper/contrail-ansible-deployer), even if you're already familiar with other CNI imlementation, since TungstenFabric uses several tools which is not in vanilla linux.
So I would recommend firstly trying the setting which works well to see what's new, and after that, integrate other systems.

Unfortunately, many repos of Tungsten Fabric are similar to rawhide and in some cases, have broken dependency.

So I picked one combination which I think mostly always works and stable enough to try most features.

To try this, you need two servers, one is for k8s master, and the other is for k8s node.
k8s master need to have at least 2 vCPUs and 8GB mem, and 8GB disk. k8s node needs 1 vCPU and 4GB mem, 8GB disk.
 - I personally always use ami-3185744e (CentOS7.5, login-id: centos) in ap-northeast-1 region, with t2.large size
 - Since in my impression, openstack and vCenter integration with Tungsten Fabric is much more complex than one with kubernetes, I recommend firstly try this setup, even if you don't need container support
 - For installation, internet connection is required

```
## all the commands are typed at k8s master node
yum -y install epel-release git ansible-2.4.2.0
ssh-keygen
cd .ssh/
cat id_rsa.pub >> authorized_keys
ssh-copy-id root@(k8s node's ip) ## or manually register id_rsa.pub to authorized_keys
cd
git clone -b R5.0 http://github.com/Juniper/contrail-ansible-deployer
cd contrail-ansible-deployer
vi config/instances.yaml
(replace contents with this)
provider_config:
  bms:
   ssh_user: root
   ssh_public_key: /root/.ssh/id_rsa.pub
   ssh_private_key: /root/.ssh/id_rsa
   domainsuffix: local
   ntpserver: 0.centos.pool.ntp.org
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
  CONTRAIL_CONTAINER_TAG: r5.0.1
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
```

One point to be taken cared of is that it is a fairly strict requirement to use supported kernel version, since Tungsten Fabric uses its own kernel module (vrouter.ko) for it's data plane.
I tried CentOS7.5, 7.6, Ubuntu Xenial and noticed it works well (for Ubuntu Bionic, some modification is needed), but if it is the first time to try, I will recommend that specific AMI id, since debuging what's not working is not an easy task.


If all the playbooks worked well, you can firstly type,
```
contrail-status
```
, which checks if everything is ok.
```
[root@ip-172-31-14-47 contrail-ansible-deployer]# contrail-status 
Pod              Service         Original Name                          State    Status             
                 redis           contrail-external-redis                running  Up 5 minutes       
analytics        alarm-gen       contrail-analytics-alarm-gen           running  Up 2 minutes       
analytics        api             contrail-analytics-api                 running  Up 2 minutes       
analytics        collector       contrail-analytics-collector           running  Up 2 minutes       
analytics        nodemgr         contrail-nodemgr                       running  Up 2 minutes       
analytics        query-engine    contrail-analytics-query-engine        running  Up 2 minutes       
analytics        snmp-collector  contrail-analytics-snmp-collector      running  Up 2 minutes       
analytics        topology        contrail-analytics-topology            running  Up 2 minutes       
config           api             contrail-controller-config-api         running  Up 4 minutes       
config           device-manager  contrail-controller-config-devicemgr   running  Up 3 minutes       
config           nodemgr         contrail-nodemgr                       running  Up 4 minutes       
config           schema          contrail-controller-config-schema      running  Up 4 minutes       
config           svc-monitor     contrail-controller-config-svcmonitor  running  Up 4 minutes       
config-database  cassandra       contrail-external-cassandra            running  Up 4 minutes       
config-database  nodemgr         contrail-nodemgr                       running  Up 4 minutes       
config-database  rabbitmq        contrail-external-rabbitmq             running  Up 4 minutes       
config-database  zookeeper       contrail-external-zookeeper            running  Up 4 minutes       
control          control         contrail-controller-control-control    running  Up 3 minutes       
control          dns             contrail-controller-control-dns        running  Up 3 minutes       
control          named           contrail-controller-control-named      running  Up 3 minutes       
control          nodemgr         contrail-nodemgr                       running  Up 3 minutes       
database         cassandra       contrail-external-cassandra            running  Up 2 minutes       
database         kafka           contrail-external-kafka                running  Up 2 minutes       
database         nodemgr         contrail-nodemgr                       running  Up 2 minutes       
database         zookeeper       contrail-external-zookeeper            running  Up 2 minutes       
kubernetes       kube-manager    contrail-kubernetes-kube-manager       running  Up About a minute  
webui            job             contrail-controller-webui-job          running  Up 3 minutes       
webui            web             contrail-controller-webui-web          running  Up 3 minutes       

WARNING: container with original name 'contrail-external-redis' have Pod or Service empty. Pod: '' / Service: 'redis'. Please pass NODE_TYPE with pod name to container's env

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
kafka: active
nodemgr: initializing (Disk for DB is too low. )
zookeeper: active
cassandra: active

== Contrail analytics ==
snmp-collector: active
query-engine: active
api: active
alarm-gen: active
nodemgr: active
collector: active
topology: active

== Contrail webui ==
web: active
job: active

== Contrail config ==
svc-monitor: active
nodemgr: active
device-manager: active
api: active
schema: active

[root@ip-172-31-14-47 contrail-ansible-deployer]# 

[root@ip-172-31-41-236 ~]# contrail-status 
Pod      Service  Original Name           State    Status         
vrouter  agent    contrail-vrouter-agent  running  Up 52 seconds  
vrouter  nodemgr  contrail-nodemgr        running  Up 52 seconds  

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: active
agent: active

[root@ip-172-31-41-236 ~]#
```


That should show most components are in 'active' state, except for
```
nodemgr: initializing (Disk for DB is too low.)
```
, which you can safely ignore in demo setup.

Note:
Which basically indicates /'s usage is over 50% and it is an important issue for cassandra.


If everything is ok, you can try this command, to see the status of Tungsten Fabric routing tables.
```
pip install lxml prettytable
git clone https://github.com/vcheny/contrail-introspect-cli.git
./contrail-introspect-cli/ist.py ctr status
./contrail-introspect-cli/ist.py ctr nei ## similar to 'show bgp summary'
./contrail-introspect-cli/ist.py ctr route summary ## similar to 'show route summary'
./contrail-introspect-cli/ist.py ctr route tables ## show routing-tables
./contrail-introspect-cli/ist.py ctr route show ## similar to 'show route'


[root@ip-172-31-14-47 contrail-ansible-deployer]# ./contrail-introspect-cli/ist.py ctr status
module_id: contrail-control
state: Functional
description
+-----------+-----------+---------------------+--------+----------------------------------+
| type      | name      | server_addrs        | status | description                      |
+-----------+-----------+---------------------+--------+----------------------------------+
| Collector | n/a       |   172.31.14.47:8086 | Up     | Established                      |
| Database  | Cassandra |   172.31.14.47:9041 | Up     | Established Cassandra connection |
| Database  | RabbitMQ  |   172.31.14.47:5673 | Up     | RabbitMQ connection established  |
+-----------+-----------+---------------------+--------+----------------------------------+

[root@ip-172-31-14-47 contrail-ansible-deployer]# ./contrail-introspect-cli/ist.py ctr nei
+--------------------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                                 | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+--------------------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-41-236.ap-                 | 172.31.41.236 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
| northeast-1.compute.internal         |               |          |          |           |             |            |            |           |
+--------------------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+

[root@ip-172-31-14-47 contrail-ansible-deployer]# ./contrail-introspect-cli/ist.py ctr route summary
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| name                                               | prefixes | paths | primary_paths | secondary_paths | infeasible_paths |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| default-domain:default-                            | 0        | 0     | 0             | 0               | 0                |
| project:__link_local__:__link_local__.inet.0       |          |       |               |                 |                  |
| default-domain:default-project:default-virtual-    | 0        | 0     | 0             | 0               | 0                |
| network:default-virtual-network.inet.0             |          |       |               |                 |                  |
| inet.0                                             | 0        | 0     | 0             | 0               | 0                |
| default-domain:default-project:ip-fabric:ip-       | 3        | 3     | 1             | 2               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 3        | 3     | 1             | 2               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 3        | 3     | 1             | 2               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+

[root@ip-172-31-14-47 contrail-ansible-deployer]# ./contrail-introspect-cli/ist.py ctr route tables
name: default-domain:default-project:__link_local__:__link_local__.inet.0
name: default-domain:default-project:default-virtual-network:default-virtual-network.inet.0
name: inet.0
name: default-domain:default-project:ip-fabric:ip-fabric.inet.0
name: default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0
name: default-domain:k8s-default:k8s-default-service-network:k8s-default-service-network.inet.0

[root@ip-172-31-14-47 contrail-ansible-deployer]# ./contrail-introspect-cli/ist.py ctr route show

bgp.ermvpn.0: 6 destinations, 6 routes (0 primary, 6 secondary, 0 infeasible)

1-172.31.41.236:1-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:02:26.545449, last_modified: 2019-Apr-13 01:41:18.023211
    [Local|None] age: 0:02:26.548569, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

1-172.31.41.236:2-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:01:09.096721, last_modified: 2019-Apr-13 01:42:35.471939
    [Local|None] age: 0:01:09.100272, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

1-172.31.41.236:3-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:00:41.812247, last_modified: 2019-Apr-13 01:43:02.756413
    [Local|None] age: 0:00:41.816037, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

2-172.31.41.236:1-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:02:26.544851, last_modified: 2019-Apr-13 01:41:18.023809
    [Local|None] age: 0:02:26.548875, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

2-172.31.41.236:2-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:01:09.096567, last_modified: 2019-Apr-13 01:42:35.472093
    [Local|None] age: 0:01:09.100828, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

2-172.31.41.236:3-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:00:41.812032, last_modified: 2019-Apr-13 01:43:02.756628
    [Local|None] age: 0:00:41.816542, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

bgp.evpn.0: 3 destinations, 3 routes (0 primary, 3 secondary, 0 infeasible)

2-172.31.41.236:1-0-0e:92:cc:bd:aa:08,0.0.0.0, age: 0:02:26.545224, last_modified: 2019-Apr-13 01:41:18.023436
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.550028, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'mpls-o-gre', 'udp'], label: 20, AS path: None

2-172.31.41.236:1-0-0e:92:cc:bd:aa:08,172.31.41.236, age: 0:02:26.545271, last_modified: 2019-Apr-13 01:41:18.023389
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.550313, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'mpls-o-gre', 'udp'], label: 20, AS path: None

3-172.31.41.236:1-2-172.31.41.236, age: 0:02:26.545365, last_modified: 2019-Apr-13 01:41:18.023295
    [Local|None] age: 0:02:26.550656, localpref: 100, nh: 172.31.41.236, encap: ['vxlan'], label: 2, AS path: None

bgp.l3vpn.0: 3 destinations, 3 routes (0 primary, 3 secondary, 0 infeasible)

172.31.41.236:1:172.31.41.236/32, age: 0:02:26.545019, last_modified: 2019-Apr-13 01:41:18.023641
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.550608, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp', 'native'], label: 16, AS path: None

172.31.41.236:2:10.47.255.252/32, age: 0:00:41.733374, last_modified: 2019-Apr-13 01:43:02.835286
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.739187, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

172.31.41.236:3:10.96.0.10/32, age: 0:00:41.732905, last_modified: 2019-Apr-13 01:43:02.835755
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.738945, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

bgp.rtarget.0: 7 destinations, 7 routes (7 primary, 0 secondary, 0 infeasible)

64512:target:64512:8000001, age: 0:02:26.592101, last_modified: 2019-Apr-13 01:41:17.976559
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.598445, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

64512:target:64512:8000002, age: 0:02:26.592073, last_modified: 2019-Apr-13 01:41:17.976587
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.598626, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

64512:target:64512:8000003, age: 0:02:26.592051, last_modified: 2019-Apr-13 01:41:17.976609
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.598800, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

64512:target:172.31.14.47:0, age: 0:05:09.194543, last_modified: 2019-Apr-13 01:38:35.374117
    [Local|None] age: 0:05:09.201488, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

64512:target:172.31.14.47:1, age: 0:02:26.592028, last_modified: 2019-Apr-13 01:41:17.976632
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.599168, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

64512:target:172.31.14.47:4, age: 0:01:09.099898, last_modified: 2019-Apr-13 01:42:35.468762
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:01:09.107253, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

64512:target:172.31.14.47:5, age: 0:00:41.824049, last_modified: 2019-Apr-13 01:43:02.744611
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.831612, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

default-domain:default-project:ip-fabric:ip-fabric.ermvpn.0: 3 destinations, 3 routes (3 primary, 0 secondary, 0 infeasible)

0-172.31.41.236:1-0.0.0.0,255.255.255.255,0.0.0.0, age: 0:02:26.544896, last_modified: 2019-Apr-13 01:41:18.023764
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.552710, localpref: 100, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 0, AS path: None

1-0:0-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:02:26.545544, last_modified: 2019-Apr-13 01:41:18.023116
    [Local|None] age: 0:02:26.553571, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

2-0:0-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:02:26.544992, last_modified: 2019-Apr-13 01:41:18.023668
    [Local|None] age: 0:02:26.553215, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

default-domain:default-project:ip-fabric:ip-fabric.evpn.0: 4 destinations, 4 routes (4 primary, 0 secondary, 0 infeasible)

2-0:0-0-0e:92:cc:bd:aa:08,0.0.0.0, age: 0:02:26.545298, last_modified: 2019-Apr-13 01:41:18.023362
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.553810, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'mpls-o-gre', 'udp'], label: 20, AS path: None

2-0:0-0-0e:92:cc:bd:aa:08,172.31.41.236, age: 0:02:26.545318, last_modified: 2019-Apr-13 01:41:18.023342
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.554076, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'mpls-o-gre', 'udp'], label: 20, AS path: None

2-172.31.41.236:1-2-ff:ff:ff:ff:ff:ff,0.0.0.0, age: 0:02:26.545486, last_modified: 2019-Apr-13 01:41:18.023174
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.554476, localpref: 100, nh: 172.31.41.236, encap: ['vxlan'], label: 2, AS path: None

3-172.31.41.236:1-2-172.31.41.236, age: 0:02:26.545411, last_modified: 2019-Apr-13 01:41:18.023249
    [Local|None] age: 0:02:26.554614, localpref: 100, nh: 172.31.41.236, encap: ['vxlan'], label: 2, AS path: None

default-domain:default-project:ip-fabric:ip-fabric.inet.0: 3 destinations, 3 routes (1 primary, 2 secondary, 0 infeasible)

10.47.255.252/32, age: 0:00:41.733312, last_modified: 2019-Apr-13 01:43:02.835348
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.742801, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

10.96.0.10/32, age: 0:00:41.732847, last_modified: 2019-Apr-13 01:43:02.835813
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.742561, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

172.31.41.236/32, age: 0:02:26.545051, last_modified: 2019-Apr-13 01:41:18.023609
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.554985, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp', 'native'], label: 16, AS path: None

default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.ermvpn.0: 3 destinations, 3 routes (3 primary, 0 secondary, 0 infeasible)

0-172.31.41.236:2-0.0.0.0,255.255.255.255,0.0.0.0, age: 0:01:09.096823, last_modified: 2019-Apr-13 01:42:35.471837
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:01:09.107020, localpref: 100, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 0, AS path: None

1-0:0-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:01:09.096765, last_modified: 2019-Apr-13 01:42:35.471895
    [Local|None] age: 0:01:09.107383, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

2-0:0-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:01:09.096621, last_modified: 2019-Apr-13 01:42:35.472039
    [Local|None] age: 0:01:09.107473, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0: 3 destinations, 3 routes (1 primary, 2 secondary, 0 infeasible)

10.47.255.252/32, age: 0:00:41.733411, last_modified: 2019-Apr-13 01:43:02.835249
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.744526, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

10.96.0.10/32, age: 0:00:41.732872, last_modified: 2019-Apr-13 01:43:02.835788
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.744256, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

172.31.41.236/32, age: 0:02:26.544986, last_modified: 2019-Apr-13 01:41:18.023674
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.556602, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp', 'native'], label: 16, AS path: None

default-domain:k8s-default:k8s-default-service-network:k8s-default-service-network.ermvpn.0: 3 destinations, 3 routes (3 primary, 0 secondary, 0 infeasible)

0-172.31.41.236:3-0.0.0.0,255.255.255.255,0.0.0.0, age: 0:00:41.812457, last_modified: 2019-Apr-13 01:43:02.756203
    [XMPP|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.824352, localpref: 100, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 0, AS path: None

1-0:0-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:00:41.812393, last_modified: 2019-Apr-13 01:43:02.756267
    [Local|None] age: 0:00:41.824504, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

2-0:0-172.31.14.47,255.255.255.255,0.0.0.0, age: 0:00:41.812099, last_modified: 2019-Apr-13 01:43:02.756561
    [Local|None] age: 0:00:41.824428, localpref: 100, nh: 172.31.14.47, encap: [], label: 0, AS path: None

default-domain:k8s-default:k8s-default-service-network:k8s-default-service-network.inet.0: 3 destinations, 3 routes (1 primary, 2 secondary, 0 infeasible)

10.47.255.252/32, age: 0:00:41.733337, last_modified: 2019-Apr-13 01:43:02.835323
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.745932, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

10.96.0.10/32, age: 0:00:41.732935, last_modified: 2019-Apr-13 01:43:02.835725
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:00:41.745758, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 25, AS path: None

172.31.41.236/32, age: 0:02:26.544959, last_modified: 2019-Apr-13 01:41:18.023701
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:02:26.558031, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp', 'native'], label: 16, AS path: None
[root@ip-172-31-14-47 contrail-ansible-deployer]# 

```


If it shows similar, since everything is working well, you can create containers based on k8s yaml.
```
vi first-containers.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: cirros-deployment
  labels:
    app: cirros-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cirros-deployment
  template:
    metadata:
      labels:
        app: cirros-deployment
    spec:
      containers:
      - name: cirros
        image: cirros
        ports:
        - containerPort: 22

kubectl create -f first-containers.yaml
kubectl get pod -o wide ## check pod name and ip
kubectl exec -it cirros-deployment-xxxx sh
ping (another pod's ip)


[root@ip-172-31-14-47 ~]# kubectl create -f first-containers.yaml
deployment "cirros-deployment" created
[root@ip-172-31-14-47 ~]# 
[root@ip-172-31-14-47 ~]# kubectl get pod -o wide
NAME                                 READY     STATUS    RESTARTS   AGE       IP              NODE
cirros-deployment-54b65ccf48-cr9dd   1/1       Running   0          34s       10.47.255.250   ip-172-31-41-236.ap-northeast-1.compute.internal
cirros-deployment-54b65ccf48-z9dds   1/1       Running   0          34s       10.47.255.251   ip-172-31-41-236.ap-northeast-1.compute.internal
[root@ip-172-31-14-47 ~]#

[root@ip-172-31-14-47 ~]# kubectl exec -it cirros-deployment-54b65ccf48-cr9dd sh
/ # 
/ # 
/ # ping 10.47.255.251
PING 10.47.255.251 (10.47.255.251): 56 data bytes
64 bytes from 10.47.255.251: seq=0 ttl=63 time=0.572 ms
64 bytes from 10.47.255.251: seq=1 ttl=63 time=0.086 ms
^C
--- 10.47.255.251 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.086/0.329/0.572 ms
/ # 
```

Cool! It will be the first packet transmitted through TungstenFabric vRouters.


If it doesn't work well, please never mind. Tungten Fabric has an active slack which could help you.
Put logs on there and try to have a help to resolve that issue.
https://tungstenfabric.slack.com


Typing 'ist.py ctr route show' again, you will see k8s-pod-network is filled with ips from two pods and next-hop for each pod is the same as k8s node's ip.
```
./contrail-introspect-cli/ist.py ctr route show (pod ip) ## similar to 'show route (some ip)'


[root@ip-172-31-14-47 contrail-ansible-deployer]# ./contrail-introspect-cli/ist.py ctr route show 10.47.255.250

default-domain:default-project:ip-fabric:ip-fabric.inet.0: 5 destinations, 5 routes (1 primary, 4 secondary, 0 infeasible)

10.47.255.250/32, age: 0:03:10.553628, last_modified: 2019-Apr-13 01:46:13.217388
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:03:10.556716, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 37, AS path: None

default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0: 5 destinations, 5 routes (3 primary, 2 secondary, 0 infeasible)

10.47.255.250/32, age: 0:03:10.553734, last_modified: 2019-Apr-13 01:46:13.217282
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:03:10.557251, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 37, AS path: None

default-domain:k8s-default:k8s-default-service-network:k8s-default-service-network.inet.0: 5 destinations, 5 routes (1 primary, 4 secondary, 0 infeasible)

10.47.255.250/32, age: 0:03:10.553654, last_modified: 2019-Apr-13 01:46:13.217362
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:03:10.557453, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 37, AS path: None
[root@ip-172-31-14-47 contrail-ansible-deployer]# 
```


Note that ip-fabric VN and k8s-default-service-network also has that prefix, since k8s-pod-network's routes are leaked to those networks.
To have route for a specific routing table, you can use -t option.
```
[root@ip-172-31-14-47 contrail-ansible-deployer]# ./contrail-introspect-cli/ist.py ctr route show -t default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0 10.47.255.251

default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0: 5 destinations, 5 routes (3 primary, 2 secondary, 0 infeasible)

10.47.255.251/32, age: 0:05:44.533377, last_modified: 2019-Apr-13 01:46:09.193202
    [XMPP (interface)|ip-172-31-41-236.ap-northeast-1.compute.internal] age: 0:05:44.536291, localpref: 200, nh: 172.31.41.236, encap: ['gre', 'udp'], label: 32, AS path: None
[root@ip-172-31-14-47 contrail-ansible-deployer]# 
```


### Appendix: external access
I think there are some misunderstandng that Tungsten Fabric always needs good routers to have external access.

It is actually not true, since from v4.1, it supports a feature called gatewayless, which allow containers directly communicate with outside world (it is also useful for similar usecase with calico)

To enable this feature, you can login Tungsten Fabric webui (https://(k8s masters's ip):8143, admin:contrail123) and reach Configure > Networks > k8s-default-pod-network, to toggle Advanced Options > IP Fabric Forwarding.

If ping from a container to k8s master ip is typed, you will notice k8s master receive a packet from container, and adding static route to k8s master, ping works well.
 - please note that you need to configure k8s node's interface setting (EC2 > Network Interfaces > Change Source/Dest Check > Disabled) if you're using AWS.

So it allows similar setting with network nodes based external access, which is based on static route on routers.

You can optionally use IPV4 bgp in combination with gatewayless, which is also recommended, since it dynamically updates the next-hops for each containers and directly send packets to the correct vRouters, which  eliminates bottleneck.

Note: this virtual-network can also be used as a source of floating-ip.
 1. Set 'Advanced Options' > 'External' to this virtual-network (Then floating ip pool will be created with the name 'default')
 2. Assign floating ip from kubernetes or openstack
 - for kubernetes,it will be the source of external-ip, and need to be specifed with this parameter to kube-manager: KUBERNETES_PUBLIC_FIP_POOL
example:
 KUBERNETES_PUBLIC_FIP_POOL={'domain': 'default-domain', 'project': 'default', 'network': 'public-network1', 'name': 'default' }
 - for openstack, horizon or cli can be used to assign floating-ip to VMs,
 3. You can also directly assign floting-ip to specific port from Tungsten Fabric Webui. (Configure > Ports > edit > floating-ip)

## 3. After this reading

Since it might be the first exposure to Tungsten Fabric, where to go after this reading is an important subject.
There are a lot of things to be worked on, such as HA, monitoring, integration with other orchestrators or router/switches, etc ...

There are a lot of resources on the web, but to pick some, I'll firstly recommend some resources from Contrail packages and education materials, even if you only will use open source version.
 - https://www.juniper.net/documentation/product/en_US/contrail-networking
 - https://www.juniper.net/uk/en/training/certification/certification-tracks/cloud-track?tab=jncia-cloud

Tungsten Fabric is a powerful platform with bunch of features, such as security-policy, analytics, l3dsr loadbalancers, service-chain, bgpaas, to name some, and many of them are non trivial features to solve real world problems.
Those links will contain a lot of contents and links to other resources.

There are also several communication channel such as mail lists and slack. Please try them if you need some help :)
https://tungsten.io/community/


# Components in Tungsten Fabric

There are a lot of different components in Tungsten Fabric.
Let me briefly describe the usage of these parts.

## Overall picture
In summary, there are 7 roles and (up to) 30 micro-services in Tungsten Fabric.
 - roles: vRouter, control, config, config-database, analytics (From 5.1, that can be further break down into analytics, analytics-snmp, analytics-alarm), analytics-database, webui

Although there are a lot of components, in simple usecase, only 4 role will be required
 - vRouter, control, config, config-database
, although in most cases, webui also will be a requirement.

You can also omit analytics if you're only interested in control-plane / data-plane part of Tungsten Fabric, although in that case, some feature (v1 service-chain, haproxy loadbalancer (and k8s ingress), SNAT etc) won't work well.

## control, vRouter

control, vRouter will be the control plane and data plane of Tungsten Fabric, so arguably, this is the most important part of Tungsten Fabric system.

Since both control and vRouter use MPLS-VPN internally, I would recommend at least skimming through this material before delving into the detail of them.
 - https://www.juniper.net/uk/en/training/certification/certification-tracks/sp-routing-switching-track?tab=jncis-sp
 - https://www.juniper.net/uk/en/training/certification/certification-tracks/sp-routing-switching-track?tab=jncip-sp
 
Since most of advanced features in control, vRouter is inherent in MPLS, those material will help to undestand what they are trying to do.

Since control and vrouter-agent uses VPNV4 bgp internally, vRouter and it's internal VRFs will install prefix needed based on extended community (a.k.a. route-target).
So when containers or vms are created on vRouter, it can signal VPNV4 route to control, and it reflects all the routes to other vRouters, and dataplane will understand where to send the packets automatically.

One interesting behavior is vRouter's virtual-network could have multiple default gateway, with same ip and same mac! (similar behavior with virtual-gateway-address, in junos's term)
Since no VRRP is required to serve default gw for each virtual-network, it eliminates the bottleneck and lets everything fully  distributed.

vRouter also is doing flow based handling for some features like statefull firewall, NAT, flow-based ECMP, ..
That is an important difference, since that behavior will introduce some tuning points, such as connection per second and maximum number of flows. (In packet based system, PPS (packet per second), and throughput (and latency in some case) will be the key)
If you're system is keen on these parameter, perhaps you need to review these parameter also.

Note: This behavior is optionally disabled with 'packet-mode' parameter in 'ports' configuration

## config (config-api, schema-transformer, svc-monitor)

Config also has several components. Config-api serves an api endpoint for Tungsten Fabric configuration, which is used many components, like control, analytics, etc
 - vRouter won't use that directly, since only the data needed is propagated from control, through xmpp

Two processes, schema-transformer and svc-monitor, are doing important things, so let me also describe them.

### schema-transformer

This process is converting some abstract config parameter, such as logical-router, network-policy, service-chain, into the words of L3VPN.
So it is one of the core components of Tungsten Fabric, and doing most of all the things which can't be explained simply by MPLS-VPN.

Logical-router, as an example, internally creates a new route-target id, which will have all the prefix connected virtual-network has. So if virtual-network is attached logical-router, it would receive all the routes logical-router has.
That behavior uses MPLS-VPN internally, but route-target configuration is controlled by schema-transformer.

So changes are propagated to dataplane in this manner:
```
edit config -> (rabbitmq) -> schema-transformer, which creates new route-target -> (internally edit config) -> (rabbitmq) -> control -> (xmpp) -> vrouter-agent -> (netlink) -> vrouter.ko
```

Schema-transformer also is doing all the things related to service-chain. I won't delve into all the detail of service chain, since that is not used simple DC usecases (even AWS VPC doesn't offer similar service currently), although internally, that's doing interesting handling of all the prefixes received around VRFs, and I personally think it is worth a read.

Note: You can have all the detail in this book.
 - https://mplsinthesdnera.net/

### svc-monitor

This process serves several services which have to use external processes internally, such as haproxy load balancer, v1 service-chain instance based on nova API, iptables MASQUERADE for SNAT, ... .

Internally, vrouter-agent has some logic to kick haproxy or set iptables MASQUERADE, svc-monitor will kick that logic, when related service is defined.

Svc-monitor chooses some vRouters to create these services, and instantiate some network function and do traffic handling to these elements. To choose one, it uses analytics-api's output (analytics/uves/vrouter), and pick one that is 'Functional'.
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py#L149

That behavior is the one reason currently analytics is required for TungstenFabric installation, although it might be changed in the future release.

## config-database (zookeeper, cassandra, rabbitmq)

Tungsten Fabric uses several databases. Most of the data are saved in cassandra, and if they are changed, rabbitmq is notified those changes to propagate other components, such as control, schema-transformer, svc-monitor, ...

Zookeeper is used only for the operation that needs lock for consistency.
For example, creating one port requires to assign one ip address, whose consistency is covered by zookeeper, so ip address assignment always will be one-by-one.

## nodemgr

I think most of the important components are covered by now, so I will cover other parts.
Firstly, let me describe what nodemgr is.

Nodemgr basically meant to be the source of the state of each node, so it checks things such as /'s usage, docker ps or cpu usage and send analytics UVE NodeStatus.
 - https://github.com/Juniper/contrail-controller/blob/master/src/nodemgr/common/linux_sys_data.py

This value could be the source of contrail-status, and other logic like analytics-alarm or svc-monitor, which check if this value is Functional when it choose vRouter, so to keep those Functional is fairly important to make Tungsten Fabric operational.

This component have a bit different behavior if assigned different role. So it is installed on each node, with slightly different behavior.

Additionaly, it also does the first provision of each nodes, which means to notify config-api that this ip has a role xxx assigned. So even if the analytics feature is not required, this module need to be there, at least for the first time a node is up.


## analytics

Tungsten Fabric analytics has a lot of features, but most of the feature is currently optional, so let me skip most of the components.
If interested, please check those links for snmp, lldp, alarms etc.
 - http://www.opencontrail.org/sandesh-a-sdn-analytics-interface/
 - http://www.opencontrail.org/operational-state-in-the-opencontrail-system-uve-user-visible-entities-through-analytics-api/
 - http://www.opencontrail.org/contrail-alerts/
 - http://www.opencontrail.org/overlay-to-physical-network-correlation/

Analytics itself has curious architecture, which covers both of logs/flows, and stats. 
 - AFAIK, those are frequently covered by different set of systems, such as EFK for logs/flows and prometheus for stats

If you need something handy for all of them, Tungsten Fabric analytics will be a good fit.

Most of the important metrics analytics serve is tagged as UVE (User Visible Entity), and have a URL to serve data with JSON format.
 - http://(analytics-ip):8081/analytics/uves has all the values available
 
If you need to integrate Tungsten Fabric with other monitoring systems, that could be a good start point.

## analytics-database

Analytics also uses several databases like redis, cassandra, kafka (internally, it also uses zookeeper for HA deployment of optional components).

If only analytics is used, redis is the only requirement and even in this setup, most of webui feature is available.
 - Most of the visualization uses UVEs so that can be available even if cassandra is not installed

Cassandra is needed if you need 'Query' feature of webui, which retrieve logs/flows or stats in cassndra db.

Kafka is used to propagate UVEs to analytics-alarms, so if you want to use alarm feature, kafka is also required.

## webui (webui-web, webui-job)

Finally, webui is reached.
It basically is a simple webui, to see the status of components and to configure parameters for Tungsten Fabric.

A bit interesting behavior is it uses AJAX behavior, to update some graph which needs long query against analytics-api (such as Monitor > Dashboard access), and that async job is covered by webui-job process.


# Orchestrator integration

Tungsten Fabric has been integrated with several orchestrators.

Internally, Tungsten Fabric's orchestrator integration components basically do the same things with each orchstrator.
 1. assign a port when vm or container is up
 2. plug that to vm or container
 
Let me describe what is done for each orchestrator.

## Openstack

When used with openstack, neutron-plugin (https://github.com/Juniper/contrail-neutron-plugin) will be the main interface between openstack and Tungsten Fabric Controller.

Neutron-plugin will be directly loaded in neutron-api process (some modules need to be specified in neutron.conf), and that logic will do things related to neutron request/response, such as network-list or port-create, and so on.

One feature of this module is that it won't use neutron db, which will be created in MySQL in typical openstack setup.

Since it directly uses Tungsten Fabric db, some features, such as bridge assignment to vm, will be a bit more difficult to achieve.
 - Since nova still uses the same vif assign logic, it might not be impossible to emulate neutron response to assign specific vif-type which can be used in neutron, although not all combination is tested, AFAIK.
 - SR-IOV is the exception of this, since emulation of that is supported and tested well
  - https://github.com/Juniper/contrail-controller/wiki/SRIOV

When a port is assigned vif-type: vrouter, which will be automatically done by 'create port' API through that neutron-plugin, it will use nova-vif-driver for vRouter (https://github.com/Juniper/contrail-nova-vif-driver), which will do some tasks other than just creating a tap device when called, such as creating vif on vRouter through vrouter-port-control script, etc.
 - In most cases, you don't need to delve into the detail of those behavior. Although in some situations like live migration stopped somewhere, you might need to be careful about the status of vif ..

## kubernetes

When used with kubernetes, the behavior is similar to openstack case, although it uses CNI for nova-vif-driver, and kube-manager for neutron-api.
 - https://github.com/Juniper/contrail-controller/tree/master/src/container/cni
 - https://github.com/Juniper/contrail-controller/tree/master/src/container/kube-manager

So when a container is created, kube-manager will create a port in Tungsten Fabric controller, and cni will assign that port to that container.


## vCenter

vCenter / Tungsten Fabric integration takes a bit different approach with kvm, since modules can't be installed directly on ESXi.

Firstly, to make the overlay available between ESXis, one vRouterVM needs to be created on each ESXi (that is a simple CentOS vm internally)

When one vm is created on that ESXi, and that was attached to dv-portgroup which was created by vcenter-plugin (https://github.com/Juniper/contrail-vcenter-plugin) when a virtual-network is created in 'vCenter' tenant, vcenter-manager (https://github.com/Juniper/contrail-vcenter-manager), which is installed on each vRouterVM with ESXi's ip / user / pass, will do two things.
 1. Set one vlan-id to the specific dv-portgroup port where that vm is attached
 2. Create a vif on the vRouterVM with interface(vlan) with the same vlan-id with that dv-portgroup port, and the VRF for that virtual-network

So when a vm sent a traffic, it will got tagged when it goes into dvswitch, and reach vRouterVM, and untagged there and go into the specific VRF, that the vm belongs to.
 - Since traffic from each vm will be tagged with different vlan-id, micro-segmentation also will be achieved

After traffic go into vRouterVM, it will be the same behavior with kvm case.

Please note that those behavior will be kicked only when vm is attached to dv-portgroups create by Tungsten Fabric controller, so vm's interfaces can be still assigned to some vSS or vDS, to use underlay access.
 - It is even possible to install vCenter and Tungsten Fabric controller to the same ESXi with vRouters (one ESXi install), if it is assigned to such as 'VM Network', rather than dv-portgroups created by Tungsten Fabric controller.


Since vRouter's behavior is the same with other cases, sharing virtual-networks between vCenter and openstack, or route leak between them are also readily available.
So with Tungsten Fabric, it is much easier to use both VMIs simultaneously, with shared networks and network services, such as fw, lb, and so on.

# More on Installation

In Up and Running chapter, I descirbed 1 controller and 1 vRouter setting, so no HA case is covered yet (And no overlay traffic case, indeed!)
Let me describe more realistic case, with 3 controllers and 2 computes (and possibly with multi-NICs) for each orchestrator.
 - In this chapter, I'll use opencontrailnightly:latest repo, since several features are not available in 5.0.1 release, but please notice that this repo could be a bit unstable in some cases.


## HA behavior of Tungsten Fabric components

When setup for serious traiffc is planned, HA always will be a requirement.

Tungsten Fabric has a decent HA implmentation, which are already documented there.
 - http://www.opencontrail.org/opencontrail-architecture-documentation/#section2_7

One thing I'd like to add is cassandra's keyspace has different replication-factor between configdb and analyticsdb.
 - configdb: https://github.com/Juniper/contrail-controller/blob/master/src/config/common/vnc_cassandra.py#L609
 - analytics: https://github.com/Juniper/contrail-analytics/blob/master/contrail-collector/db_handler.cc#L524

Since configdb's data is replicated to all cassandras, it is fairly unlikely to lose some data, even if some node's disk has crashed and needs to be wiped out.
On the other hand, since analyticsdb's replication-factor is always two, if two nodes lost data simultaneously, the data could be lost.

## Multi-NIC installation

When installing Tungsten Fabric, there are many situations that requires multi-nic installation, such as separate NIC for management plane and control / data plane.
 - Bonding is not included in this discussion, since bond0 can be directly specified by VROUTER_GATEWAY parameter

Let me clarify curious behavior of vRouter in this setup.

For controller / analytics, that won't be much different from typical linux installation, since linux will work well with multiple NICs and its own routing-table, including the use of static route.

On the other hand, in vRouter nodes, you need to be a bit careful, since vRouter won't use linux routing-table when it sends packets, rather it always sends packets to one and only one gateway ip.
 - It can be set with gateway parameter in contrail-vrouter-agent.conf, and VROUTER_GATEWAY in vrouter-agent container's environment variable

So when setting up multi-nic installation, you need to be a bit careful if you need to specify VROUTER_GATEWAY.

If it doesn't specified, vrouter-agent container will pick the nic that holds default route of that node, although that won't be the correct NIC, if internet access (0.0.0.0/0) is covered by management NIC, rather than data plane NIC.

In such situation, you need to explicitly specify VROUTER_GATEWAY parameter.

Because of those behavior, you also need to be a bit careful when you want to send packets from vms or containers to the NICs other than one NIC vRouter uses, since it also doesn't check linux routing-table, and it always uses the same NIC with other vRouter traffic.
 - AFAIK, packets from link-local serivce or gatewayless also show similar behavior

In such situation, you might need to use simple-gateway or SR-IOV.
 - https://github.com/Juniper/contrail-controller/wiki/Simple-Gateway


## kubeadm

When I'm writing this document, ansible-deployer haven't yet supported k8s master HA.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1761137

Since kubeadm already supports k8s master HA, I'll describe the way to integrate kubeadm based k8s install and YAML based Tungsten Fabric install.
 - https://kubernetes.io/docs/setup/independent/high-availability/
 - https://github.com/Juniper/contrail-ansible-deployer/wiki/Provision-Contrail-Kubernetes-Cluster-in-Non-nested-Mode

As other CNIs, Tungsten Fabric also can be installed directly by 'kubectl apply' command. But to achieve this, you need to configure some parameters, such as IP addr of controller nodes, manually.

For this example setup, I used 5 EC2 instances (AMI is the same, ami-3185744e). 2 vcpu, 8 GB mem, 20 GB disk is assigned to those instances. VPC has CIDR with 172.31.0.0/16

```
(on all nodes)
# cat <<CONTENTS > install-k8s-packages.sh
bash -c 'cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
     https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF'
setenforce 0
yum install -y kubelet kubeadm kubectl docker
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
swapoff -a
CONTENTS

# bash install-k8s-packages.sh

(on the first k8s master node)
yum -y install haproxy
# vi /etc/haproxy/haproxy.cfg
(add those lines at the last of this file)
listen kube
  mode tcp
  bind 0.0.0.0:1443
  server master1 172.31.13.9:6443
  server master2 172.31.8.73:6443
  server master3 172.31.32.58:6443
# systemctl start haproxy
# systemctl enable haproxy

# vi kubeadm-config.yaml 
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - "ip-172-31-13-9"
controlPlaneEndpoint: "ip-172-31-13-9:1443"

# kubeadm init --config=kubeadm-config.yaml

(save those lines for later use)
  kubeadm join ip-172-31-13-9:1443 --token mlq9gw.gt5m13cbro6c8xsu \
    --discovery-token-ca-cert-hash sha256:677ea74fa03311a38ecb497d2f0803a5ea1eea85765aa2daa4503f24dd747f9a \
    --experimental-control-plane
  kubeadm join ip-172-31-13-9:1443 --token mlq9gw.gt5m13cbro6c8xsu \
    --discovery-token-ca-cert-hash sha256:677ea74fa03311a38ecb497d2f0803a5ea1eea85765aa2daa4503f24dd747f9a 

# mkdir -p $HOME/.kube
# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config

# cd /etc/kubernetes
# tar czvf /tmp/k8s-master-ca.tar.gz pki/ca.crt pki/ca.key pki/sa.key pki/sa.pub pki/front-proxy-ca.crt pki/front-proxy-ca.key pki/etcd/ca.crt pki/etcd/ca.key admin.conf
(scp that tar file to 2nd and 3rd k8s master node)

(On 2nd and 3rd k8s master nodes)
# mkdir -p /etc/kubernetes/pki/etcd
# cd /etc/kubernetes
# tar xvf /tmp/k8s-master-ca.tar.gz

# kubeadm join ip-172-31-13-9:1443 --token mlq9gw.gt5m13cbro6c8xsu \
    --discovery-token-ca-cert-hash sha256:677ea74fa03311a38ecb497d2f0803a5ea1eea85765aa2daa4503f24dd747f9a \
    --experimental-control-plane

(on k8s nodes)
 - type kubeadm join commands, which is previosly saved
# kubeadm join ip-172-31-13-9:1443 --token mlq9gw.gt5m13cbro6c8xsu \
    --discovery-token-ca-cert-hash sha256:677ea74fa03311a38ecb497d2f0803a5ea1eea85765aa2daa4503f24dd747f9a

(on the first k8s master node)
# vi set-label.sh
masternodes=$(kubectl get node | grep -w master | awk '{print $1}')
agentnodes=$(kubectl get node | grep -v -w -e master -e NAME | awk '{print $1}')
for i in config configdb analytics webui control
do
 for masternode in ${masternodes}
 do
  kubectl label node ${masternode} node-role.opencontrail.org/${i}=
 done
done

for i in ${agentnodes}
do
 kubectl label node ${i} node-role.opencontrail.org/agent=
done

# bash set-label.sh



# yum -y install git
# git clone https://github.com/Juniper/contrail-container-builder.git
# cd /root/contrail-container-builder/kubernetes/manifests
# cat <<EOF > ../../common.env
CONTRAIL_CONTAINER_TAG=latest
CONTRAIL_REGISTRY=opencontrailnightly
EOF

# ./resolve-manifest.sh contrail-standalone-kubernetes.yaml > cni-tungsten-fabric.yaml 
# vi cni-tungsten-fabric.yaml
(manually modify those lines)
 - lines which includes ANALYTICS_API_VIP, CONFIG_API_VIP, VROUTER_GATEWAY need to be deleted
 - Several lines which include ANALYTICS_NODES. ANALYTICSDB_NODES, CONFIG_NODES, CONFIGDB_NODES, CONTROL_NODES, CONTROLLER_NODES, RABBITMQ_NODES, ZOOKEEPER_NODES need to be set properly, like CONFIG_NODES: ip1,ip2,ip3
# kubectl apply -f cni-tungsten-fabric.yaml
```

I'll attach original and modified yaml file for further reference.
 - https://github.com/tnaganawa/tungstenfabric-docs/cni-tungsten-fabric.yaml.orig
 - https://github.com/tnaganawa/tungstenfabric-docs/cni-tungsten-fabric.yaml

Then you finally have kubernetes environment with TungstenFabric CNI fully up!

```
[root@ip-172-31-13-9 ~]# kubectl get node -o wide
NAME                                               STATUS     ROLES    AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
ip-172-31-13-9.ap-northeast-1.compute.internal     NotReady   master   34m   v1.14.1   172.31.13.9     <none>        CentOS Linux 7 (Core)   3.10.0-862.2.3.el7.x86_64   docker://1.13.1
ip-172-31-17-120.ap-northeast-1.compute.internal   Ready      <none>   30m   v1.14.1   172.31.17.120   <none>        CentOS Linux 7 (Core)   3.10.0-862.2.3.el7.x86_64   docker://1.13.1
ip-172-31-32-58.ap-northeast-1.compute.internal    NotReady   master   32m   v1.14.1   172.31.32.58    <none>        CentOS Linux 7 (Core)   3.10.0-862.2.3.el7.x86_64   docker://1.13.1
ip-172-31-5-235.ap-northeast-1.compute.internal    Ready      <none>   30m   v1.14.1   172.31.5.235    <none>        CentOS Linux 7 (Core)   3.10.0-862.2.3.el7.x86_64   docker://1.13.1
ip-172-31-8-73.ap-northeast-1.compute.internal     NotReady   master   31m   v1.14.1   172.31.8.73     <none>        CentOS Linux 7 (Core)   3.10.0-862.2.3.el7.x86_64   docker://1.13.1
[root@ip-172-31-13-9 ~]# kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                                                      READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
kube-system   config-zookeeper-d897f                                                    1/1     Running   0          7m14s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   config-zookeeper-fvnbq                                                    1/1     Running   0          7m14s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   config-zookeeper-t5vjc                                                    1/1     Running   0          7m14s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-agent-cqpxc                                                      2/2     Running   0          7m12s   172.31.17.120   ip-172-31-17-120.ap-northeast-1.compute.internal   <none>           <none>
kube-system   contrail-agent-pv7c8                                                      2/2     Running   0          7m12s   172.17.0.1      ip-172-31-5-235.ap-northeast-1.compute.internal    <none>           <none>
kube-system   contrail-analytics-cfcx8                                                  3/3     Running   0          7m14s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-analytics-h5jbr                                                  3/3     Running   0          7m14s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-analytics-wvc5n                                                  3/3     Running   0          7m14s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   contrail-config-database-nodemgr-7f5h5                                    1/1     Running   0          7m14s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-config-database-nodemgr-bkmpz                                    1/1     Running   0          7m14s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-config-database-nodemgr-z6qx9                                    1/1     Running   0          7m14s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   contrail-configdb-5vd8t                                                   1/1     Running   0          7m14s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   contrail-configdb-kw6v7                                                   1/1     Running   0          7m14s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-configdb-vjv2b                                                   1/1     Running   0          7m14s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-controller-config-dk78j                                          5/5     Running   0          7m13s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-controller-config-jrh27                                          5/5     Running   0          7m14s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   contrail-controller-config-snxnn                                          5/5     Running   0          7m13s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-controller-control-446v8                                         4/4     Running   0          7m14s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-controller-control-fzpwz                                         4/4     Running   0          7m14s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   contrail-controller-control-tk52v                                         4/4     Running   1          7m14s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-controller-webui-94s26                                           2/2     Running   0          7m13s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   contrail-controller-webui-bdzbj                                           2/2     Running   0          7m13s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-controller-webui-qk4ww                                           2/2     Running   0          7m13s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-kube-manager-g6vsg                                               1/1     Running   0          7m12s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-kube-manager-ppjkf                                               1/1     Running   0          7m12s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   contrail-kube-manager-rjpmw                                               1/1     Running   0          7m12s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   coredns-fb8b8dccf-wmdw2                                                   0/1     Running   2          34m     10.47.255.252   ip-172-31-17-120.ap-northeast-1.compute.internal   <none>           <none>
kube-system   coredns-fb8b8dccf-wsrtl                                                   0/1     Running   2          34m     10.47.255.251   ip-172-31-17-120.ap-northeast-1.compute.internal   <none>           <none>
kube-system   etcd-ip-172-31-13-9.ap-northeast-1.compute.internal                       1/1     Running   0          33m     172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   etcd-ip-172-31-32-58.ap-northeast-1.compute.internal                      1/1     Running   0          32m     172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   etcd-ip-172-31-8-73.ap-northeast-1.compute.internal                       1/1     Running   0          30m     172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-apiserver-ip-172-31-13-9.ap-northeast-1.compute.internal             1/1     Running   0          33m     172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-apiserver-ip-172-31-32-58.ap-northeast-1.compute.internal            1/1     Running   1          32m     172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   kube-apiserver-ip-172-31-8-73.ap-northeast-1.compute.internal             1/1     Running   1          30m     172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-controller-manager-ip-172-31-13-9.ap-northeast-1.compute.internal    1/1     Running   1          33m     172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-controller-manager-ip-172-31-32-58.ap-northeast-1.compute.internal   1/1     Running   0          31m     172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   kube-controller-manager-ip-172-31-8-73.ap-northeast-1.compute.internal    1/1     Running   0          31m     172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-proxy-6ls9w                                                          1/1     Running   0          32m     172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   kube-proxy-82jl8                                                          1/1     Running   0          30m     172.31.5.235    ip-172-31-5-235.ap-northeast-1.compute.internal    <none>           <none>
kube-system   kube-proxy-bjdj9                                                          1/1     Running   0          31m     172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-proxy-nd7hq                                                          1/1     Running   0          31m     172.31.17.120   ip-172-31-17-120.ap-northeast-1.compute.internal   <none>           <none>
kube-system   kube-proxy-rb7nk                                                          1/1     Running   0          34m     172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-scheduler-ip-172-31-13-9.ap-northeast-1.compute.internal             1/1     Running   1          33m     172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   kube-scheduler-ip-172-31-32-58.ap-northeast-1.compute.internal            1/1     Running   0          31m     172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   kube-scheduler-ip-172-31-8-73.ap-northeast-1.compute.internal             1/1     Running   0          31m     172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   rabbitmq-9lp4n                                                            1/1     Running   0          7m12s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   rabbitmq-lxkgz                                                            1/1     Running   0          7m12s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   rabbitmq-wfk2f                                                            1/1     Running   0          7m12s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
kube-system   redis-h2x2b                                                               1/1     Running   0          7m13s   172.31.13.9     ip-172-31-13-9.ap-northeast-1.compute.internal     <none>           <none>
kube-system   redis-pkmng                                                               1/1     Running   0          7m13s   172.31.8.73     ip-172-31-8-73.ap-northeast-1.compute.internal     <none>           <none>
kube-system   redis-r68ks                                                               1/1     Running   0          7m13s   172.31.32.58    ip-172-31-32-58.ap-northeast-1.compute.internal    <none>           <none>
[root@ip-172-31-13-9 ~]# 


[root@ip-172-31-13-9 ~]# contrail-status 
Pod              Service         Original Name                          State    Id            Status             
                 redis           contrail-external-redis                running  8f38c94fc370  Up About a minute  
analytics        api             contrail-analytics-api                 running  2edde00b4525  Up About a minute  
analytics        collector       contrail-analytics-collector           running  c1d0c24775a6  Up About a minute  
analytics        nodemgr         contrail-nodemgr                       running  4a4c455cc0df  Up About a minute  
config           api             contrail-controller-config-api         running  b855ad79ace4  Up About a minute  
config           device-manager  contrail-controller-config-devicemgr   running  50d590e6f6cf  Up About a minute  
config           nodemgr         contrail-nodemgr                       running  6f0f64f958d9  Up About a minute  
config           schema          contrail-controller-config-schema      running  2057b21f50b7  Up About a minute  
config           svc-monitor     contrail-controller-config-svcmonitor  running  ba48df5cb7f9  Up About a minute  
config-database  cassandra       contrail-external-cassandra            running  1d38278d304e  Up About a minute  
config-database  nodemgr         contrail-nodemgr                       running  8e4f9315cc38  Up About a minute  
config-database  rabbitmq        contrail-external-rabbitmq             running  4a424e2f456c  Up About a minute  
config-database  zookeeper       contrail-external-zookeeper            running  4b46c83f1376  Up About a minute  
control          control         contrail-controller-control-control    running  17e4b9b9e3b8  Up About a minute  
control          dns             contrail-controller-control-dns        running  39fc34e19e13  Up About a minute  
control          named           contrail-controller-control-named      running  aef0bf56a0e2  Up About a minute  
control          nodemgr         contrail-nodemgr                       running  21f091df35d5  Up About a minute  
kubernetes       kube-manager    contrail-kubernetes-kube-manager       running  db661ef685b0  Up About a minute  
webui            job             contrail-controller-webui-job          running  0bf35b774aac  Up About a minute  
webui            web             contrail-controller-webui-web          running  9213ce050547  Up About a minute  

== Contrail control ==
control: active
nodemgr: active
named: active
dns: active

== Contrail config-database ==
nodemgr: active
zookeeper: active
rabbitmq: active
cassandra: active

== Contrail kubernetes ==
kube-manager: backup

== Contrail analytics ==
nodemgr: active
api: active
collector: active

== Contrail webui ==
web: active
job: active

== Contrail config ==
svc-monitor: backup
nodemgr: active
device-manager: active
api: active
schema: backup

[root@ip-172-31-13-9 ~]# 

[root@ip-172-31-8-73 ~]# contrail-status 
Pod              Service         Original Name                          State    Id            Status             
                 redis           contrail-external-redis                running  39af38401d31  Up 2 minutes       
analytics        api             contrail-analytics-api                 running  29fa05f18927  Up 2 minutes       
analytics        collector       contrail-analytics-collector           running  994bffbe4c1f  Up About a minute  
analytics        nodemgr         contrail-nodemgr                       running  1eb143c7b864  Up About a minute  
config           api             contrail-controller-config-api         running  92ee8983bc81  Up About a minute  
config           device-manager  contrail-controller-config-devicemgr   running  7f9ab5d2a9ca  Up About a minute  
config           nodemgr         contrail-nodemgr                       running  c6a88b487031  Up About a minute  
config           schema          contrail-controller-config-schema      running  1fe2e2767dca  Up About a minute  
config           svc-monitor     contrail-controller-config-svcmonitor  running  ec1d66894036  Up About a minute  
config-database  cassandra       contrail-external-cassandra            running  80f394c8d1a8  Up 2 minutes       
config-database  nodemgr         contrail-nodemgr                       running  af9b70285564  Up About a minute  
config-database  rabbitmq        contrail-external-rabbitmq             running  edae18a7cf9f  Up 2 minutes       
config-database  zookeeper       contrail-external-zookeeper            running  f00c2e5d94ac  Up 2 minutes       
control          control         contrail-controller-control-control    running  6e3e22625a50  Up About a minute  
control          dns             contrail-controller-control-dns        running  b1b6b9649761  Up About a minute  
control          named           contrail-controller-control-named      running  f8aa237fca10  Up About a minute  
control          nodemgr         contrail-nodemgr                       running  bb0868390322  Up About a minute  
kubernetes       kube-manager    contrail-kubernetes-kube-manager       running  02e99f8b9490  Up About a minute  
webui            job             contrail-controller-webui-job          running  f5ffdfc1076f  Up About a minute  
webui            web             contrail-controller-webui-web          running  09c3f77223d3  Up About a minute  

== Contrail control ==
control: active
nodemgr: active
named: active
dns: active

== Contrail config-database ==
nodemgr: active
zookeeper: active
rabbitmq: active
cassandra: active

== Contrail kubernetes ==
kube-manager: backup

== Contrail analytics ==
nodemgr: active
api: active
collector: active

== Contrail webui ==
web: active
job: active

== Contrail config ==
svc-monitor: backup
nodemgr: active
device-manager: backup
api: active
schema: backup

[root@ip-172-31-8-73 ~]# 

[root@ip-172-31-32-58 ~]# contrail-status 
Pod              Service         Original Name                          State    Id            Status             
                 redis           contrail-external-redis                running  44363e63f104  Up 2 minutes       
analytics        api             contrail-analytics-api                 running  aa8c5dc17c57  Up 2 minutes       
analytics        collector       contrail-analytics-collector           running  6856b8e33f34  Up 2 minutes       
analytics        nodemgr         contrail-nodemgr                       running  c1ec67695618  Up About a minute  
config           api             contrail-controller-config-api         running  ff95a8e3e4a9  Up 2 minutes       
config           device-manager  contrail-controller-config-devicemgr   running  abc0ad6b32c0  Up 2 minutes       
config           nodemgr         contrail-nodemgr                       running  c883e525205a  Up About a minute  
config           schema          contrail-controller-config-schema      running  0b18780b02da  Up About a minute  
config           svc-monitor     contrail-controller-config-svcmonitor  running  42e74aad3d3d  Up About a minute  
config-database  cassandra       contrail-external-cassandra            running  3994d9f51055  Up 2 minutes       
config-database  nodemgr         contrail-nodemgr                       running  781c5c93e662  Up 2 minutes       
config-database  rabbitmq        contrail-external-rabbitmq             running  849427f37237  Up 2 minutes       
config-database  zookeeper       contrail-external-zookeeper            running  fbb778620915  Up 2 minutes       
control          control         contrail-controller-control-control    running  85b2e8366a13  Up 2 minutes       
control          dns             contrail-controller-control-dns        running  b1f05dc6b8ee  Up 2 minutes       
control          named           contrail-controller-control-named      running  ca68ff0e118b  Up About a minute  
control          nodemgr         contrail-nodemgr                       running  cf8aaff71343  Up About a minute  
kubernetes       kube-manager    contrail-kubernetes-kube-manager       running  62022a542509  Up 2 minutes       
webui            job             contrail-controller-webui-job          running  28413e9f378b  Up 2 minutes       
webui            web             contrail-controller-webui-web          running  4a6edac6d596  Up 2 minutes       

== Contrail control ==
control: active
nodemgr: active
named: active
dns: active

== Contrail config-database ==
nodemgr: active
zookeeper: active
rabbitmq: active
cassandra: active

== Contrail kubernetes ==
kube-manager: active

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
device-manager: backup
api: active
schema: active

[root@ip-172-31-32-58 ~]# 

[root@ip-172-31-5-235 ~]# contrail-status 
Pod      Service  Original Name           State    Id            Status        
vrouter  agent    contrail-vrouter-agent  running  48377d29f584  Up 2 minutes  
vrouter  nodemgr  contrail-nodemgr        running  77d7a409d410  Up 2 minutes  

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: active
agent: active

[root@ip-172-31-5-235 ~]# 

[root@ip-172-31-17-120 ~]# contrail-status 
Pod      Service  Original Name           State    Id            Status        
vrouter  agent    contrail-vrouter-agent  running  f97837959a0b  Up 3 minutes  
vrouter  nodemgr  contrail-nodemgr        running  4e48673efbcc  Up 3 minutes  

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: active
agent: active


[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py ctr nei
+--------------------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------------------------+
| peer                                 | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time                   |
+--------------------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------------------------+
| ip-172-31-32-58.ap-                  | 172.31.32.58  | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a                         |
| northeast-1.compute.internal         |               |          |          |           |             |            |            |                             |
| ip-172-31-8-73.ap-                   | 172.31.8.73   | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a                         |
| northeast-1.compute.internal         |               |          |          |           |             |            |            |                             |
| ip-172-31-17-120.ap-                 | 172.31.17.120 | 0        | XMPP     | internal  | Established | in sync    | 5          | 2019-Apr-28 07:35:40.743648 |
| northeast-1.compute.internal         |               |          |          |           |             |            |            |                             |
| ip-172-31-5-235.ap-                  | 172.31.5.235  | 0        | XMPP     | internal  | Established | in sync    | 6          | 2019-Apr-28 07:35:40.251476 |
| northeast-1.compute.internal         |               |          |          |           |             |            |            |                             |
+--------------------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------------------------+
[root@ip-172-31-13-9 ~]# 


[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py ctr route summary
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| name                                               | prefixes | paths | primary_paths | secondary_paths | infeasible_paths |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| default-domain:default-                            | 0        | 0     | 0             | 0               | 0                |
| project:__link_local__:__link_local__.inet.0       |          |       |               |                 |                  |
| default-domain:default-project:dci-                | 0        | 0     | 0             | 0               | 0                |
| network:__default__.inet.0                         |          |       |               |                 |                  |
| default-domain:default-project:dci-network:dci-    | 0        | 0     | 0             | 0               | 0                |
| network.inet.0                                     |          |       |               |                 |                  |
| default-domain:default-project:default-virtual-    | 0        | 0     | 0             | 0               | 0                |
| network:default-virtual-network.inet.0             |          |       |               |                 |                  |
| inet.0                                             | 0        | 0     | 0             | 0               | 0                |
| default-domain:default-project:ip-fabric:ip-       | 4        | 8     | 2             | 6               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 4        | 8     | 2             | 6               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 4        | 8     | 0             | 8               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]# 
```

After the cirros deployment is created just like Up and Running section, ping between two vRouter nodes will be available.

```
[root@ip-172-31-13-9 ~]# kubectl get pod -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE   READINESS GATES
cirros-deployment-86885fbf85-pkzqz   1/1     Running   0          16s   10.47.255.249   ip-172-31-17-120.ap-northeast-1.compute.internal   <none>           <none>
cirros-deployment-86885fbf85-w4w6h   1/1     Running   0          16s   10.47.255.250   ip-172-31-5-235.ap-northeast-1.compute.internal    <none>           <none>
[root@ip-172-31-13-9 ~]# 
[root@ip-172-31-13-9 ~]# 
[root@ip-172-31-13-9 ~]# kubectl exec -it cirros-deployment-86885fbf85-pkzqz sh
/ # ping 10.47.255.250
PING 10.47.255.250 (10.47.255.250): 56 data bytes
64 bytes from 10.47.255.250: seq=0 ttl=63 time=3.376 ms
64 bytes from 10.47.255.250: seq=1 ttl=63 time=2.587 ms
64 bytes from 10.47.255.250: seq=2 ttl=63 time=2.549 ms
^C
--- 10.47.255.250 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 2.549/2.837/3.376 ms
/ # 
/ # 
/ # ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
23: eth0@if24: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue \    link/ether 02:64:0d:41:b0:69 brd ff:ff:ff:ff:ff:ff
23: eth0    inet 10.47.255.249/12 scope global eth0\       valid_lft forever preferred_lft forever
23: eth0    inet6 fe80::489a:28ff:fedf:2e7b/64 scope link \       valid_lft forever preferred_lft forever
/ # 

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py ctr route summary
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| name                                               | prefixes | paths | primary_paths | secondary_paths | infeasible_paths |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| default-domain:default-                            | 0        | 0     | 0             | 0               | 0                |
| project:__link_local__:__link_local__.inet.0       |          |       |               |                 |                  |
| default-domain:default-project:dci-                | 0        | 0     | 0             | 0               | 0                |
| network:__default__.inet.0                         |          |       |               |                 |                  |
| default-domain:default-project:dci-network:dci-    | 0        | 0     | 0             | 0               | 0                |
| network.inet.0                                     |          |       |               |                 |                  |
| default-domain:default-project:default-virtual-    | 0        | 0     | 0             | 0               | 0                |
| network:default-virtual-network.inet.0             |          |       |               |                 |                  |
| inet.0                                             | 0        | 0     | 0             | 0               | 0                |
| default-domain:default-project:ip-fabric:ip-       | 6        | 12    | 2             | 10              | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 6        | 12    | 4             | 8               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 6        | 12    | 0             | 12              | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]# 

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py ctr route show -t default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0 10.47.255.251

default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0: 6 destinations, 12 routes (4 primary, 8 secondary, 0 infeasible)

10.47.255.251/32, age: 0:08:37.590508, last_modified: 2019-Apr-28 07:37:16.031523
    [XMPP (interface)|ip-172-31-17-120.ap-northeast-1.compute.internal] age: 0:08:37.596128, localpref: 200, nh: 172.31.17.120, encap: ['gre', 'udp'], label: 25, AS path: None
    [BGP|172.31.32.58] age: 0:08:37.594533, localpref: 200, nh: 172.31.17.120, encap: ['gre', 'udp'], label: 25, AS path: None


[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py ctr route show -t default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0 10.47.255.250

default-domain:k8s-default:k8s-default-pod-network:k8s-default-pod-network.inet.0: 6 destinations, 12 routes (4 primary, 8 secondary, 0 infeasible)

10.47.255.250/32, age: 0:01:50.135045, last_modified: 2019-Apr-28 07:44:06.371447
    [XMPP (interface)|ip-172-31-5-235.ap-northeast-1.compute.internal] age: 0:01:50.141480, localpref: 200, nh: 172.31.5.235, encap: ['gre', 'udp'], label: 25, AS path: None
    [BGP|172.31.32.58] age: 0:01:50.098328, localpref: 200, nh: 172.31.5.235, encap: ['gre', 'udp'], label: 25, AS path: None
[root@ip-172-31-13-9 ~]# 
```



## Openstack

Openstack HA installation is directly covered by ansible-deployer.

For this example setup, I used 5 EC2 instances (AMI is the same, ami-3185744e).
2 vcpu, 8 GB mem, 20 GB disk is assigned to those instances. VPC has CIDR with 172.31.0.0/16.

```
yum -y install epel-release
yum -y install git ansible-2.4.2.0
ssh-keygen
cd .ssh/
cat id_rsa.pub >> authorized_keys
cd
git clone http://github.com/Juniper/contrail-ansible-deployer
cd contrail-ansible-deployer
vi config/instances.yaml
(replace contents with this)
provider_config:
  bms:
   ssh_user: root
   ssh_public_key: /root/.ssh/id_rsa.pub
   ssh_private_key: /root/.ssh/id_rsa
   domainsuffix: local
   ntpserver: 0.centos.pool.ntp.org
instances:
  bms1:
    provider: bms
    ip: 172.31.6.90 # controller1's ip
    roles:
      config_database:
      config:
      control:
      analytics:
      webui:
      openstack:
  bms2:
    provider: bms
    ip: 172.31.25.90 # controller2's ip
    roles:
      config_database:
      config:
      control:
      analytics:
      webui:
      openstack:
  bms3:
    provider: bms
    ip: 172.31.31.242 # controller3's ip
    roles:
      config_database:
      config:
      control:
      analytics:
      webui:
      openstack:
  bms11:
    provider: bms
    ip: 172.31.42.209 # compute1's ip
    roles:
      vrouter:
      openstack_compute:
  bms12:
    provider: bms
    ip: 172.31.15.199 # compute2's ip
    roles:
      vrouter:
      openstack_compute:
contrail_configuration:
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
  JVM_EXTRA_OPTS: "-Xms128m -Xmx1g"
kolla_config:
  kolla_globals:
    kolla_internal_vip_address: 172.31.0.11 ## kolla-ansible will deploy haproxy to serve HA vip
  kolla_passwords:
    keystone_admin_password: contrail123 # admin user's password
global_configuration:


## if previously described AMI is used, it uses cloud-init packages whose rpm dependency is not compatible with ansible-deployer in R5.1 and later. To workaroud this, I used these commands.
yum -y remove PyYAML python-requests
pip install PyYAML requests
pip install ansible


ansible-playbook -e orchestrator=openstack -i inventory/ playbooks/configure_instances.yml
 - it takes about 10 minutes
ansible-playbook -e orchestrator=openstack -i inventory/ playbooks/install_openstack.yml
 - it takes about 40 minutes
ansible-playbook -e orchestrator=openstack -i inventory/ playbooks/install_contrail.yml
 - it takes about 20 minutes



[root@ip-172-31-6-90 ~]# contrail-status 
Pod              Service         Original Name                          State    Id            Status         
                 redis           contrail-external-redis                running  23ef79b48ae8  Up 41 minutes  
analytics        api             contrail-analytics-api                 running  3139f5fd9256  Up 36 minutes  
analytics        collector       contrail-analytics-collector           running  89c9e02fb551  Up 36 minutes  
analytics        nodemgr         contrail-nodemgr                       running  5eecb461f95c  Up 36 minutes  
config           api             contrail-controller-config-api         running  fb0dc55f76c7  Up 39 minutes  
config           device-manager  contrail-controller-config-devicemgr   running  8dbff58776a2  Up 39 minutes  
config           nodemgr         contrail-nodemgr                       running  b64af838545d  Up 39 minutes  
config           schema          contrail-controller-config-schema      running  83e0acf17e39  Up 39 minutes  
config           svc-monitor     contrail-controller-config-svcmonitor  running  623e17e8e74e  Up 39 minutes  
config-database  cassandra       contrail-external-cassandra            running  db30d874dce3  Up 40 minutes  
config-database  nodemgr         contrail-nodemgr                       running  590463f627f6  Up 38 minutes  
config-database  rabbitmq        contrail-external-rabbitmq             running  712ee26dda64  Up 40 minutes  
config-database  zookeeper       contrail-external-zookeeper            running  46dbdec00e46  Up 40 minutes  
control          control         contrail-controller-control-control    running  3e0e653d1588  Up 37 minutes  
control          dns             contrail-controller-control-dns        running  2cebc37c18cf  Up 37 minutes  
control          named           contrail-controller-control-named      running  112bd2d8ed5f  Up 37 minutes  
control          nodemgr         contrail-nodemgr                       running  f2e0fdc4bfb2  Up 37 minutes  
device-manager   dnsmasq         contrail-external-dnsmasq              running  f84b45234d70  Up 39 minutes  
webui            job             contrail-controller-webui-job          running  3dece86513a1  Up 38 minutes  
webui            web             contrail-controller-webui-web          running  408c772b1628  Up 38 minutes  

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

== Contrail analytics ==
nodemgr: active
api: active
collector: active

== Contrail webui ==
web: active
job: active

== Contrail device-manager ==

== Contrail config ==
svc-monitor: backup
nodemgr: active
device-manager: backup
api: active
schema: backup


[root@ip-172-31-25-90 ~]# contrail-status 
Pod              Service         Original Name                          State    Id            Status         
                 redis           contrail-external-redis                running  1ed7e967085e  Up 41 minutes  
analytics        api             contrail-analytics-api                 running  7392ea345e83  Up 36 minutes  
analytics        collector       contrail-analytics-collector           running  82332a53a566  Up 36 minutes  
analytics        nodemgr         contrail-nodemgr                       running  89141bb180cd  Up 36 minutes  
config           api             contrail-controller-config-api         running  b2af8bc8a6d7  Up 38 minutes  
config           device-manager  contrail-controller-config-devicemgr   running  d8ed77431dfa  Up 39 minutes  
config           nodemgr         contrail-nodemgr                       running  8c7f3d5f05e4  Up 39 minutes  
config           schema          contrail-controller-config-schema      running  4a6099aaea2a  Up 39 minutes  
config           svc-monitor     contrail-controller-config-svcmonitor  running  3a3e6d37b30e  Up 39 minutes  
config-database  cassandra       contrail-external-cassandra            running  0b05e121c017  Up 40 minutes  
config-database  nodemgr         contrail-nodemgr                       running  fb4857fe16c1  Up 39 minutes  
config-database  rabbitmq        contrail-external-rabbitmq             running  a8137277a40f  Up 40 minutes  
config-database  zookeeper       contrail-external-zookeeper            running  9571f4d9fde2  Up 40 minutes  
control          control         contrail-controller-control-control    running  5460dc02cc03  Up 37 minutes  
control          dns             contrail-controller-control-dns        running  17b27877ef6e  Up 37 minutes  
control          named           contrail-controller-control-named      running  cdbe1bae4c40  Up 37 minutes  
control          nodemgr         contrail-nodemgr                       running  cb36c2b4625a  Up 37 minutes  
device-manager   dnsmasq         contrail-external-dnsmasq              running  dd9002e6f58d  Up 39 minutes  
webui            job             contrail-controller-webui-job          running  60dc895d439e  Up 38 minutes  
webui            web             contrail-controller-webui-web          running  3ddfb5e2e851  Up 38 minutes  

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

== Contrail analytics ==
nodemgr: active
api: active
collector: active

== Contrail webui ==
web: active
job: active

== Contrail device-manager ==

== Contrail config ==
svc-monitor: backup
nodemgr: active
device-manager: active
api: active
schema: backup


[root@ip-172-31-31-242 ~]# contrail-status 
Pod              Service         Original Name                          State    Id            Status         
                 redis           contrail-external-redis                running  172e35daca5a  Up 42 minutes  
analytics        api             contrail-analytics-api                 running  2edf90837a43  Up 36 minutes  
analytics        collector       contrail-analytics-collector           running  812d4c190841  Up 36 minutes  
analytics        nodemgr         contrail-nodemgr                       running  d0eafce0d49d  Up 36 minutes  
config           api             contrail-controller-config-api         running  7819c7792960  Up 39 minutes  
config           device-manager  contrail-controller-config-devicemgr   running  c22addf8f1f1  Up 38 minutes  
config           nodemgr         contrail-nodemgr                       running  bd742928f26e  Up 39 minutes  
config           schema          contrail-controller-config-schema      running  8ad72d0a2c12  Up 39 minutes  
config           svc-monitor     contrail-controller-config-svcmonitor  running  86283bfc21dc  Up 39 minutes  
config-database  cassandra       contrail-external-cassandra            running  315d17494665  Up 41 minutes  
config-database  nodemgr         contrail-nodemgr                       running  a78521b2b940  Up 39 minutes  
config-database  rabbitmq        contrail-external-rabbitmq             running  dfefb054808b  Up 41 minutes  
config-database  zookeeper       contrail-external-zookeeper            running  a16d1a2d259b  Up 41 minutes  
control          control         contrail-controller-control-control    running  bc9ecb41131c  Up 37 minutes  
control          dns             contrail-controller-control-dns        running  beff8cf11fdd  Up 37 minutes  
control          named           contrail-controller-control-named      running  2322d5598a24  Up 37 minutes  
control          nodemgr         contrail-nodemgr                       running  32b611d85d19  Up 37 minutes  
device-manager   dnsmasq         contrail-external-dnsmasq              running  a0b3dd0ad254  Up 39 minutes  
webui            job             contrail-controller-webui-job          running  257721b46207  Up 38 minutes  
webui            web             contrail-controller-webui-web          running  c2e7b95e7321  Up 38 minutes  

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

== Contrail analytics ==
nodemgr: active
api: active
collector: active

== Contrail webui ==
web: active
job: active

== Contrail device-manager ==

== Contrail config ==
svc-monitor: active
nodemgr: active
device-manager: backup
api: active
schema: active


[root@ip-172-31-42-209 ~]# contrail-status 
Pod      Service  Original Name           State    Id            Status         
vrouter  agent    contrail-vrouter-agent  running  a17883037f12  Up 36 minutes  
vrouter  nodemgr  contrail-nodemgr        running  6dc2258ac4f6  Up 36 minutes  

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: active
agent: active


[root@ip-172-31-15-199 ~]# contrail-status 
Pod      Service  Original Name           State    Id            Status         
vrouter  agent    contrail-vrouter-agent  running  a1e7767b3302  Up 36 minutes  
vrouter  nodemgr  contrail-nodemgr        running  40d5613fec21  Up 36 minutes  

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: active
agent: active
```


Then, you can create instances with openstack command.

```
docker cp /etc/kolla/kolla-toolbox/admin-openrc.sh kolla_toolbox:/var/tmp
docker exec -it kolla_toolbox bash
  source /var/tmp/admin-openrc.sh
  cd /var/tmp
  curl -O http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
  openstack image create cirros --disk-format qcow2 --public --container-format bare --file cirros-0.4.0-x86_64-disk.img
  openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny
  openstack network create testvn
  openstack subnet create --subnet-range 192.168.100.0/24 --network testvn subnet1
  NET_ID=`openstack network list | grep testvn | awk -F '|' '{print $2}' | tr -d ' '`
  openstack server create --flavor m1.tiny --image cirros --nic net-id=${NET_ID} vm1
  openstack server create --flavor m1.tiny --image cirros --nic net-id=${NET_ID} vm2
  exit

(on compute nodes)
ip route ## check metadata ip of two instances
ssh cirros@169.254.0.x
  ping 192.168.100.4


(kolla-toolbox)[ansible@ip-172-31-6-90 /]$ openstack server list
+--------------------------------------+------+--------+----------------------+--------+---------+
| ID                                   | Name | Status | Networks             | Image  | Flavor  |
+--------------------------------------+------+--------+----------------------+--------+---------+
| 9d66f0ed-d7d5-4a53-983d-dfba0385bd22 | vm2  | ACTIVE | testvn=192.168.100.4 | cirros | m1.tiny |
| 6595b4c1-1e6f-4f02-8f66-83b6355065b2 | vm1  | ACTIVE | testvn=192.168.100.3 | cirros | m1.tiny |
+--------------------------------------+------+--------+----------------------+--------+---------+
(kolla-toolbox)[ansible@ip-172-31-6-90 /]$ 

[root@ip-172-31-42-209 ~]# ip route
default via 172.31.32.1 dev vhost0 
169.254.0.1 dev vhost0 proto 109 scope link 
169.254.0.3 dev vhost0 proto 109 scope link 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
172.31.32.0/20 dev vhost0 proto kernel scope link src 172.31.42.209 
[root@ip-172-31-42-209 ~]# ssh cirros@169.254.0.3
cirros@169.254.0.3's password: 
$ ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000\    link/ether 02:79:59:ea:d4:17 brd ff:ff:ff:ff:ff:ff
2: eth0    inet 192.168.100.3/24 brd 192.168.100.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::79:59ff:feea:d417/64 scope link \       valid_lft forever preferred_lft forever
$ 
$ ping 192.168.100.4
PING 192.168.100.4 (192.168.100.4): 56 data bytes
64 bytes from 192.168.100.4: seq=0 ttl=64 time=13.876 ms
64 bytes from 192.168.100.4: seq=1 ttl=64 time=2.417 ms
64 bytes from 192.168.100.4: seq=2 ttl=64 time=2.375 ms
^C
--- 192.168.100.4 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 2.375/6.222/13.876 ms
$ 
$


[root@ip-172-31-15-199 ~]# ip route
default via 172.31.0.1 dev vhost0 
169.254.0.1 dev vhost0 proto 109 scope link 
169.254.0.3 dev vhost0 proto 109 scope link 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
172.31.0.0/20 dev vhost0 proto kernel scope link src 172.31.15.199 
[root@ip-172-31-15-199 ~]# ssh cirros@169.254.0.3
cirros@169.254.0.3's password: 
$ ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000\    link/ether 02:08:e6:0d:1e:3b brd ff:ff:ff:ff:ff:ff
2: eth0    inet 192.168.100.4/24 brd 192.168.100.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::8:e6ff:fe0d:1e3b/64 scope link \       valid_lft forever preferred_lft forever
$ 
```


Note: you might need this setting to be added, if compute nodes don't support kvm.
```
vi /etc/kolla/nova-compute/nova.conf
(add them in [libvirt] section)
virt_type=qemu
cpu_mode=none

docker restart nova_compute
```

Note: If AWS is used, you also need to set Networking > Manage IP Addresses from EC2 instance's right click menu, to allow access to the haproxy VIP from other nodes


Finally, full HA between controllers and overlay between 2 computes are configured!

There are some points that is not covered in this document, such as the behavior when some controllers are down, or live migration between them are performed between computes.
When I tried live migration last time, about 1 sec packet loss is seen, but please check in your own setup, since there are a lot of points to be taken cared of. (prefix will be updated when live migration is finished)


Looking through each controller's neighbor state and routing table, you can see curious difference between them.

```
[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py ctr nei
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-25-90.local  | 172.31.25.90  | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-31-242.local | 172.31.31.242 | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-42-209.local | 172.31.42.209 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py --host 172.31.25.90 ctr nei
Introspect Host: 172.31.25.90
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-31-242.local | 172.31.31.242 | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-6-90.local   | 172.31.6.90   | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-15-199.local | 172.31.15.199 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-6-90 ~]# 
[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py --host 172.31.31.242 ctr nei
Introspect Host: 172.31.31.242
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-25-90.local  | 172.31.25.90  | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-6-90.local   | 172.31.6.90   | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-15-199.local | 172.31.15.199 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-42-209.local | 172.31.42.209 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-6-90 ~]#
[root@ip-172-31-6-90 ~]#

[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py ctr route summary
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| name                                               | prefixes | paths | primary_paths | secondary_paths | infeasible_paths |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| default-domain:admin:testvn:testvn.inet.0          | 2        | 4     | 1             | 3               | 0                |
| default-domain:default-                            | 0        | 0     | 0             | 0               | 0                |
| project:__link_local__:__link_local__.inet.0       |          |       |               |                 |                  |
| default-domain:default-project:dci-                | 0        | 0     | 0             | 0               | 0                |
| network:__default__.inet.0                         |          |       |               |                 |                  |
| default-domain:default-project:dci-network:dci-    | 0        | 0     | 0             | 0               | 0                |
| network.inet.0                                     |          |       |               |                 |                  |
| default-domain:default-project:default-virtual-    | 0        | 0     | 0             | 0               | 0                |
| network:default-virtual-network.inet.0             |          |       |               |                 |                  |
| inet.0                                             | 0        | 0     | 0             | 0               | 0                |
| default-domain:default-project:ip-fabric:ip-       | 1        | 1     | 1             | 0               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py --host 172.31.25.90 ctr route summary
Introspect Host: 172.31.25.90
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| name                                               | prefixes | paths | primary_paths | secondary_paths | infeasible_paths |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| default-domain:admin:testvn:testvn.inet.0          | 2        | 4     | 1             | 3               | 0                |
| default-domain:default-                            | 0        | 0     | 0             | 0               | 0                |
| project:__link_local__:__link_local__.inet.0       |          |       |               |                 |                  |
| default-domain:default-project:dci-                | 0        | 0     | 0             | 0               | 0                |
| network:__default__.inet.0                         |          |       |               |                 |                  |
| default-domain:default-project:dci-network:dci-    | 0        | 0     | 0             | 0               | 0                |
| network.inet.0                                     |          |       |               |                 |                  |
| default-domain:default-project:default-virtual-    | 0        | 0     | 0             | 0               | 0                |
| network:default-virtual-network.inet.0             |          |       |               |                 |                  |
| inet.0                                             | 0        | 0     | 0             | 0               | 0                |
| default-domain:default-project:ip-fabric:ip-       | 1        | 1     | 1             | 0               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py --host 172.31.31.242 ctr route summary
Introspect Host: 172.31.31.242
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| name                                               | prefixes | paths | primary_paths | secondary_paths | infeasible_paths |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| default-domain:admin:testvn:testvn.inet.0          | 2        | 4     | 2             | 2               | 0                |
| default-domain:default-                            | 0        | 0     | 0             | 0               | 0                |
| project:__link_local__:__link_local__.inet.0       |          |       |               |                 |                  |
| default-domain:default-project:dci-                | 0        | 0     | 0             | 0               | 0                |
| network:__default__.inet.0                         |          |       |               |                 |                  |
| default-domain:default-project:dci-network:dci-    | 0        | 0     | 0             | 0               | 0                |
| network.inet.0                                     |          |       |               |                 |                  |
| default-domain:default-project:default-virtual-    | 0        | 0     | 0             | 0               | 0                |
| network:default-virtual-network.inet.0             |          |       |               |                 |                  |
| inet.0                                             | 0        | 0     | 0             | 0               | 0                |
| default-domain:default-project:ip-fabric:ip-       | 2        | 2     | 2             | 0               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-6-90 ~]#

 
[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py --host 172.31.31.242 ctr route show 192.168.100.3
Introspect Host: 172.31.31.242

default-domain:admin:testvn:testvn.inet.0: 2 destinations, 4 routes (2 primary, 2 secondary, 0 infeasible)

192.168.100.3/32, age: 0:01:18.234010, last_modified: 2019-Apr-27 14:03:19.075046
    [XMPP (interface)|ip-172-31-42-209.local] age: 0:01:18.239011, localpref: 200, nh: 172.31.42.209, encap: ['gre', 'udp'], label: 25, AS path: None
    [BGP|172.31.6.90] age: 0:01:18.230559, localpref: 200, nh: 172.31.42.209, encap: ['gre', 'udp'], label: 25, AS path: None


[root@ip-172-31-6-90 ~]# ./contrail-introspect-cli/ist.py --host 172.31.31.242 ctr route show 192.168.100.4
Introspect Host: 172.31.31.242

default-domain:admin:testvn:testvn.inet.0: 2 destinations, 4 routes (2 primary, 2 secondary, 0 infeasible)

192.168.100.4/32, age: 0:00:52.035230, last_modified: 2019-Apr-27 14:03:47.460835
    [XMPP (interface)|ip-172-31-15-199.local] age: 0:00:52.039485, localpref: 200, nh: 172.31.15.199, encap: ['gre', 'udp'], label: 25, AS path: None
    [BGP|172.31.25.90] age: 0:00:51.996464, localpref: 200, nh: 172.31.15.199, encap: ['gre', 'udp'], label: 25, AS path: None
[root@ip-172-31-6-90 ~]# 
```

Since vRouter always have 2 XMPP connections, when 3 controllers are there, XMPP connection states are not the same between controllers, and routing tables could be a bit different between them.
Considering route target filtering, it is even possible that they have completely different routing tables, if some of controllers don't received some specific route-target from XMPP.

That is from the nature of scale-out behavior of Tungsten Fabric.



For more detailed configuration of ansible-deployer (including multi-NIC sample), you can check those documents.
 - https://github.com/Juniper/contrail-ansible-deployer/wiki/Contrail-with-Openstack-Kolla
 - https://github.com/Juniper/contrail-ansible-deployer/wiki/Configuration-Sample-for-Multi-Node-Openstack-HA-and-Contrail-(single-interface)
 - https://github.com/Juniper/contrail-ansible-deployer/wiki/Configuration-Sample-for-Multi-Node-Openstack-HA-and-Contrail-(multi-interface)

## vCenter

Tungsten Fabric can be well integrated with vCenter, as already described orchestrator integration section.

To try this feature, you can follow these instructions.
 - https://github.com/Juniper/contrail-ansible-deployer/blob/master/README_vcenter.md
 - Let me note that vRouterVM's OVF file is not publicly available currently

Since HA behavior of Tungsten Fabric is completely the same as the one from kubernetes and openstack installation, I don't describe the detail of them.

For orchestrator side HA, vCenter HA is available.
 - I haven't yet tried this combination, but since vCenter HA will use the same IP for vCenter service, I think there is high possibility that vcenter-plugin can work with vCenter HA.

Multi-vCenter or cross-vCenter (when link-mode is used) will be a bit interesting subject. I will discuss them further in Appendix.

# Monitoring integration

Although Tungsten Fabric has decent monitoring / alarm features, it could be a requirement to integrate them to full-fledged monitoring systems.

Let me describe how to integrate them with promethesus and EFK, as an example.

## Prometheus

To monitor and visualize what's going on in Tungsten Fabric systems, prometheus will be one possible choice.
 - Several tools, such as zabbix, support scraping prometheus format, so this also could be useful as a common format among monitoring tools: https://www.zabbix.com/documentation/4.2/manual/config/items/itemtypes/prometheus

To scraped by prometheus, Tungsten Fabric's metrics need to be exported in prometheus exporter format, and there are two ways to achieve this.
 1. Directly export metrics from introspect HTTP Server (this feature is not availale today)
 2. Export values from analytics, or from analytics UVE (it could be available today)

As the first step, I tried a short script (WIP) to export values to prometheus.
 - Currently, limited number of vRouters's metrics, like number of packets, number of bytes, number of flows, number of drop packets are exported
 - https://github.com/tnaganawa/tf-analytics-exporter

Those values also can be used to send alerts from prometheus, rather than from analytics-alarms.

## EFK

# Day 2 operation
## ist.py
 operational command
## contrail-api-cli
 show configuration
## webui
 configuration command

# Appendix

## L3VPN / EVPN (T2/T5, VXLAN/MPLS) integration

## Service-Chain (L2, L3, NAT), BGPaaS

## Multicluster 
### routing / bridging, DNS, security-policy
### Inter AS option B/C

## Multi orchestrator

### k8s+openstack
works well
 - https://github.com/Juniper/contrail-ansible-deployer/wiki/Deployment-Example:-Contrail-and-Kubernetes-and-Openstack

### k8s+k8s
cluster_names can be changed, but currently doesn't seem to work well ..
 - https://github.com/Juniper/contrail-container-builder/blob/master/containers/kubernetes/kube-manager/entrypoint.sh#L28

### openstack+openstack
to be investigated

### k8s+vCenter
should work well, not tried yet

### openstack+vCenter
curios, since vCenter could have tenancy / fwaas v2 / lbaas integrated

### vCenter+vCenter
multi-vcenter, to be investigated
 - might be achieved, since multiple vcenter-plugins might consume events from vcenter in parallel

### k8s+openstack+vCenter

## Service Mesh
istio is working well, multicluster could be interesting subject
 - https://www.youtube.com/watch?v=VSNc9qd2poA
 - https://istio.io/docs/setup/kubernetes/install/multicluster/vpn/
