
Table of Contents
=================

   * [A Tungsten Fabric Primer](#a-tungsten-fabric-primer)
      * [1. Why Tungsten Fabric?](#1-why-tungsten-fabric)
         * [I. Interoperability with ASIC](#i-interoperability-with-asic)
         * [II. Scalability](#ii-scalability)
      * [2. TungstenFabric, Up and Running](#2-tungstenfabric-up-and-running)
         * [Appendix: external access](#appendix-external-access)
      * [3. After this reading](#3-after-this-reading)
   * [components in Tungsten Faric](#components-in-tungsten-faric)
      * [overall picture](#overall-picture)
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
      * [openstack](#openstack)
      * [vCenter](#vcenter)
      * [kubernetes](#kubernetes)
   * [HA Installation](#ha-installation)
      * [HA behaivor of Tungsten Fabric components](#ha-behaivor-of-tungsten-fabric-components)
      * [kubeadm](#kubeadm)
      * [openstack](#openstack-1)
      * [vCenter, to be investigated for vCenter HA part](#vcenter-to-be-investigated-for-vcenter-ha-part)
   * [Monitoring integration](#monitoring-integration)
      * [prometheus](#prometheus)
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
Route target filtering feature makes that behaivor possible, and dramatically reduce the number of prefixes each vRouter (and each controller if RR is used between them) needs to take care of, which makes this control plane much more scalable.


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


# components in Tungsten Faric

There are a lot of different components in Tungsten Fabric.
Let me briefly describe the usage of these parts.

## overall picture
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

Since control and vrouter-agent uses VPNV4 bgp internally, vRouter and it's internal VRFs will install prefix needed based on extended community (a.k.a route-target).
So when containers of vms are created on vRouter, it can signal VPNV4 route to control, and it reflects all the routes to other vRouters, and dataplane will understand where to send the packets automatically.

One interesting behaivor is vRouter's virtual-network could have multiple default gateway, with same ip and same mac! (similar behaivor with virtual-gateway-address, in junos's term)
Since no VRRP is required to serve default gw for each virtual-network, it eliminates the bottleneck and lets everything fully  distributed.

vRouter also is doing flow based handling for some features like statefull firewall, NAT, flow-based ECMP, ..
That is an important difference, since that behaivor will introduce some tuning points, such as connection per second and maximum number of flows. (In packet based system, PPS (packet per second), and throughput (and latency in some case) will be the key)
If you're system is keen on these parameter, perhaps you need to review these parameter also.

Note: This behaivor is optionally disabled with 'packet-mode' parameter in 'ports' configuration

## config (config-api, schema-transformer, svc-monitor)

Config also has several components. Config-api serves an api endpoint for Tungsten Fabric configuration, which is used many components, like control, analytics, etc
 - vRouter won't use that directly, since only the data needed is propagated from control, through xmpp

Two processes, schema-transformer and svc-monitor, are doing important things, so let me also describe them.

## schema-transformer

This process is converting some abstract config parameter, such as logical-router, network-policy, service-chain, into the words of L3VPN.
So it is one of the core components of Tungsten Fabric, and doing most of all the things which can't be explained simply by MPLS-VPN.

Logical-router, as an example, internally creates a new route-target id, which will have all the prefix connected virtual-network has. So if virtual-network is attached logical-router, it would receive all the routes logical-router has.
That behaivor uses MPLS-VPN internally, but route-target configuration is controlled by schema-transformer.

So changes are propagated to dataplane in this manner:
```
edit config -> (rabbitmq) -> schema-transformer, which creates new route-target -> (internally edit config) -> (rabbitmq) -> control -> (xmpp) -> vrouter-agent -> (netlink) -> vrouter.ko
```

Schema-transformer also is doing all the things related to service-chain. I won't delve into all the detail of service chain, since that is not used simple DC usecases (even AWS VPC doesn't offer similar service currently), although internally, that's doing interesting handling of all the prefixes received around VRFs, and I personally think it is worth a read.

Note: You can have all the detail in this book.
 - https://mplsinthesdnera.net/

## svc-monitor

This process serves several services which have to use external processes internally, such as haproxy load balancer, v1 service-chain instance based on nova API, iptables MASQUERADE for SNAT, ... .

Internally, vrouter-agent has some logic to kick haproxy or set iptables MASQUERADE, svc-monitor will kick that logic, when related service is defined.

Svc-monitor chooses some vRouters to create these services, and instantiate some network function and do traffic handling to these elements. To choose one, it uses analytics-api's output (analytics/uves/vrouter), and pick one that is 'Functional'.
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py#L149

That behaivor is the one reason currently analytics is required for TungstenFabric installation, although it might be changed in the future release.

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

This component have a bit different behaivor if assigned different role. So it is installed on each node, with slightly different behaivor.

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

A bit interesting behaivor is it uses AJAX behaivor, to update some graph which needs long query against analytics-api (such as Monitor > Dashboard access), and that async job is covered by webui-job process.


# Orchestrator integration

Tungsten Fabric has been integrated with several orchestrators.

Internally, Tungsten Fabric's orchestrator integration components basically do the same things with each orchstrator.
 1. assign a port when vm or container is up
 2. plug that to vm or container
 
Let me describe what is done for each orchestrator.

## openstack
neutron plugin sends the event to Tungsten Fabric controller, and nova-compute plug that to vm

## vCenter
vcenter-plugin receive events from vCenter and create a port. vcenter-manager receive events from ESXi, and set vRouter and vDS.

## kubernetes

kubemanager receive events from kube-apiserver and create a port. cni plug that to container

# HA Installation

## HA behaivor of Tungsten Fabric components

## kubeadm
k8s master HA, and standalone, or non-nested install based on YAML

## openstack
ansible-deployer

## vCenter, to be investigated for vCenter HA part
ansible-deployer

# Monitoring integration

example to integrate other monitoring systems

## prometheus
xxx-exporter is a new word for nagios plugin ..

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

### k8s+k8s
### openstack+openstack

### k8s+vCenter
### openstack+vCenter
### vCenter+vCenter

### k8s+openstack+vCenter

## Service Mesh
