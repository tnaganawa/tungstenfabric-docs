
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
      * [device-manager](#device-manager)
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
      * [Sizing the cluster](#sizing-the-cluster)
      * [kubeadm](#kubeadm)
      * [Openstack](#openstack-1)
      * [vCenter](#vcenter-1)
      * [Container tag to be used](#container-tag-to-be-used)
   * [Monitoring integration](#monitoring-integration)
      * [Prometheus](#prometheus)
      * [EFK](#efk)
      * [Topology view](#topology-view)
   * [Day 2 operation](#day-2-operation)
      * [ist.py](#istpy)
      * [contrail-api-cli](#contrail-api-cli)
      * [webui](#webui)
      * [backup and restore](#backup-and-restore)
      * [Changing container parameters](#changing-container-parameters)
   * [Troubleshooting Tips](#troubleshooting-tips)
   * [Appendix](#appendix)
      * [Cluster update](#cluster-update)
      * [L3VPN / EVPN (T2/T5, VXLAN/MPLS) integration](#l3vpn--evpn-t2t5-vxlanmpls-integration)
      * [Service-Chain (L2, L3, NAT), BGPaaS](#service-chain-l2-l3-nat-bgpaas)
      * [Multicluster](#multicluster)
         * [Inter AS option B/C](#inter-as-option-bc)
      * [Multi DC](#multi-dc)
      * [Multi orchestrator](#multi-orchestrator)
         * [k8s openstack](#k8sopenstack)
         * [k8s k8s](#k8sk8s)
         * [openstack openstack](#openstackopenstack)
         * [k8s vCenter](#k8svcenter)
         * [openstack vCenter](#openstackvcenter)
         * [vCenter vCenter](#vcentervcenter)
         * [k8s openstack vCenter](#k8sopenstackvcenter)
      * [DPDK](#dpdk)
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

To try TungstenFabric for the first time, I recommend using ansible-deployer (https://github.com/Juniper/contrail-ansible-deployer), even if you're already familiar with other CNI implementation, since TungstenFabric uses several tools which is not in vanilla linux.
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
yum -y remove PyYAML python-requests
yum -y install git
easy_install pip
pip install PyYAML requests ansible==2.7.11
ssh-keygen
cd .ssh/
cat id_rsa.pub >> authorized_keys
ssh-copy-id root@(k8s node's ip) ## or manually register id_rsa.pub to authorized_keys
cd
git clone -b R5.1 http://github.com/Juniper/contrail-ansible-deployer
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
  CONTRAIL_CONTAINER_TAG: r5.1
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
 - You also need to set a network policy between that VN and default-domain:default-project:ip-fabric, since without that, RPF check will drop that packet

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

## device-manager

This process is used to configure physical-router, based on objects in config-database.

Internally, it uses the same logic with schema-transformer and svc-monitor, which subscribe rabbitmq to see config change, and when something changed, the amqp client kicks some logic.
 - For schema-transformer, it will update some more config, for svc-monitor, it will kick some logic in vRouters, and for device-manager, it will update physical-router's configuration

This behavior is controlled by reaction_map, which defines the way some change on some config objects will propagate other config's change.
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/device-manager/device_manager/device_manager.py#L59

For example, when a bgp-router is updated,
```
        'bgp_router': {
            'self': ['bgp_router', 'physical_router'],
            'bgp_router': ['physical_router'],
            'physical_router': [],
           },
```
based on 'self' definition, it will propagate to bgp-router and physical-router with refs to the original bgp-router object.
 - For bgp-router, it mean bgp-router object which peered with the original bgp-router

After that, updated bgp-router in turn propagate that to the physical-router, which bgp-router objects resides on.
```
        'bgp_router': {
            (snip)
            'bgp_router': ['physical_router'],
            (snip)
           },
```
Since physical-router won't update something when event is propagated from bgp-router, event stopped there, and physical-router config with original bgp-router, and peered bgp-router will be updated.
```
        'physical_router': {
            (snip)
            'bgp_router': [],
            (snip)
},
```

When physical-router receives update event, it will call push_conf function from plugin, it will basically create router config based on objects in config-database.
 - currently, only MX / QFX has opensource plugin: https://github.com/Juniper/contrail-controller/tree/master/src/config/device-manager/device_manager/plugins/juniper

To enable this feature, this knob is needed to be configured in /etc/contrail/common_config.env: DEVICE_MANAGER__DEFAULTS__push_mode=0, and config procedure is described there: https://www.juniper.net/documentation/en_US/contrail5.0/topics/concept/using-device-manager-netconf-contrail.html

## analytics

Tungsten Fabric analytics has a lot of features, but most of the feature is currently optional, so let me skip most of the components.
If interested, please check those links for snmp, lldp, alarms etc.
 - https://tungsten.io/sandesh-a-sdn-analytics-interface/
 - https://tungsten.io/operational-state-in-the-opencontrail-system-uve-user-visible-entities-through-analytics-api/
 - https://tungsten.io/contrail-alerts/
 - https://tungsten.io/overlay-to-physical-network-correlation/

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

Note: One recent addition is tungsten fabric also has got ml2 based plugin.
 - https://www.youtube.com/watch?v=4MkkMRR9U2s
 - https://opendev.org/x/networking-opencontrail

So if users already use ml2 with MySQL, they can firstly add vRouter as one of ml2 network-type, use that in specific virtual-network, and migrate from other ml2 plugin to vRouter by detach and attach interface. (optionally to replace neutron core plugin, if all the migration finished)

Some installation detail is also added.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricKnowledgeBase.md#vrouter-ml2-plugin

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

In Up and Running section, I descirbed 1 controller and 1 vRouter setting, so no HA case is covered yet (And no overlay traffic case, indeed!)
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

## Sizing the cluster

For general sizing of Tungsten Fabric cluster, you can use this table.
 - https://github.com/hartmutschroeder/contrailandrhosp10#21sizing-the-controller-nodes-and-vms

If cluster size is large, you need good amount of resources to serve stable control plane.

Please note that from R5.1, analytics database (and some components of analytics) become optional, so I would recommend using R5.1 release, if you want to use control plane only from Tungsten Fabric.
 - https://github.com/Juniper/contrail-analytics/blob/master/specs/analytics_optional_components.md

How large a cluster can be also is an important subject, although I don't have a handy answer, since it depends a lot of factors.
 - I once tried nearly 5,000 nodes with one k8s cluster (https://kubernetes.io/docs/setup/cluster-large/). It worked well with one controller node with 64vCPUs, 58GB mem, although at that time, I haven't created much ports, policies, and logical-routers, etc.
 - This wiki also has some real world experience about gigantic cluster: https://wiki.tungsten.io/display/TUN/KubeCon+NA+in+Seattle+2018

Since you can instantly got a lot of resources from cloud, perhaps the best option is to emulate the cluster with the size and traffic what you need, and see if it works ok and what will be the bottleneck.

Tungsten Fabric has several good features to be gigantic, such as multi-cluster setup based on MP-BGP between clusters, and BUM drop feature based on L3-only virtual-network, which could be a key to have scalable and stable virtual-network.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1471637

To illustrate control's scale-out behavior, I created a cluster with 980 vRouters and 15 controls in AWS.
 - All the control nodes have 4vCPUs and 16GB mem

```
Following instances.yaml is used to provision controller nodes,
and non-nested.yaml for kubeadm (https://github.com/Juniper/contrail-container-builder/blob/master/kubernetes/manifests/contrail-non-nested-kubernetes.yaml) is used to provision vRouters

(venv) [root@ip-172-31-21-119 ~]# cat contrail-ansible-deployer/config/instances.yaml
provider_config:
  bms:
   ssh_user: centos
   ssh_public_key: /root/.ssh/id_rsa.pub
   ssh_private_key: /tmp/aaa.pem
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
      webui:
   ip: 172.31.21.119
  bms2:
   provider: bms
   roles:
     control:
     analytics:
   ip: 172.31.21.78
  bms3:
   provider: bms

 ...

  bms13:
   provider: bms
   roles:
     control:
     analytics:
   ip: 172.31.14.189
  bms14:
   provider: bms
   roles:
     control:
     analytics:
   ip: 172.31.2.159
  bms15:
   provider: bms
   roles:
     control:
     analytics:
   ip: 172.31.7.239
contrail_configuration:
  CONTRAIL_CONTAINER_TAG: r5.1
  KUBERNETES_CLUSTER_PROJECT: {}
  JVM_EXTRA_OPTS: "-Xms128m -Xmx1g"
global_configuration:
  CONTAINER_REGISTRY: tungstenfabric
(venv) [root@ip-172-31-21-119 ~]#

[root@ip-172-31-4-80 ~]# kubectl get node | head
NAME                                               STATUS     ROLES    AGE     VERSION
ip-172-31-0-112.ap-northeast-1.compute.internal    Ready      <none>   9m24s   v1.15.0
ip-172-31-0-116.ap-northeast-1.compute.internal    Ready      <none>   9m37s   v1.15.0
ip-172-31-0-133.ap-northeast-1.compute.internal    Ready      <none>   9m37s   v1.15.0
ip-172-31-0-137.ap-northeast-1.compute.internal    Ready      <none>   9m24s   v1.15.0
ip-172-31-0-141.ap-northeast-1.compute.internal    Ready      <none>   9m24s   v1.15.0
ip-172-31-0-142.ap-northeast-1.compute.internal    Ready      <none>   9m24s   v1.15.0
ip-172-31-0-151.ap-northeast-1.compute.internal    Ready      <none>   9m37s   v1.15.0
ip-172-31-0-163.ap-northeast-1.compute.internal    Ready      <none>   9m37s   v1.15.0
ip-172-31-0-168.ap-northeast-1.compute.internal    Ready      <none>   9m16s   v1.15.0
[root@ip-172-31-4-80 ~]# 
[root@ip-172-31-4-80 ~]# kubectl get node | grep -w Ready | wc -l
980
[root@ip-172-31-4-80 ~]# 


(venv) [root@ip-172-31-21-119 ~]# contrail-api-cli --host 172.31.21.119 ls virtual-router | wc -l
980
(venv) [root@ip-172-31-21-119 ~]#
```

When number of control nodes are 15, number of XMPP connections are up 113, so CPU usage is not so high (up to 5.4%).

```
[root@ip-172-31-21-119 ~]# ./contrail-introspect-cli/ist.py ctr nei | grep -w XMPP | wc -l
113
[root@ip-172-31-21-119 ~]#

top - 05:52:14 up 42 min,  1 user,  load average: 1.73, 5.50, 3.57
Tasks: 154 total,   1 running, 153 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.4 us,  2.9 sy,  0.0 ni, 94.6 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem : 15233672 total,  8965420 free,  2264516 used,  4003736 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 12407304 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                
32368 root      20   0  839848  55240  11008 S   7.6  0.4   0:21.40 contrail-collec                                                                                        
28773 root      20   0 1311252 132552  14540 S   5.3  0.9   1:40.72 contrail-contro                                                                                        
17129 polkitd   20   0   56076  22496   1624 S   3.7  0.1   0:11.42 redis-server                                                                                           
32438 root      20   0  248496  40336   5328 S   2.0  0.3   0:15.80 python                                                                                                 
18346 polkitd   20   0 2991576 534452  22992 S   1.7  3.5   4:56.90 java                                                                                                   
15344 root      20   0  972324  97248  35360 S   1.3  0.6   2:25.84 dockerd                                                                                                
15351 root      20   0 1477100  32988  12532 S   0.7  0.2   0:08.72 docker-containe                                                                                        
18365 centos    20   0 5353996 131388   9288 S   0.7  0.9   0:09.49 java                                                                                                   
19994 polkitd   20   0 3892836 127772   3644 S   0.7  0.8   1:34.55 beam.smp                                                                                               
17112 root      20   0    7640   3288   2456 S   0.3  0.0   0:00.24 docker-containe                                                                                        
24723 root      20   0  716512  68920   6288 S   0.3  0.5   0:01.75 node     
```

However, when 12 control nodes are stoppped, number of XMPP connections per one control will be as high as 708, so CPU usage become pretty high (21.6%).

So if you need to provision fairly large number of nodes, number of control nodes might need to be planned carefully.

```
[root@ip-172-31-21-119 ~]# ./contrail-introspect-cli/ist.py ctr nei | grep -w BGP
| ip-172-31-13-119.local               | 172.31.13.119 | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:10:47.527354 |
| ip-172-31-13-87.local                | 172.31.13.87  | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:10:08.610734 |
| ip-172-31-14-189.local               | 172.31.14.189 | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:16:34.953311 |
| ip-172-31-14-243.local               | 172.31.14.243 | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:06:12.379006 |
| ip-172-31-17-212.local               | 172.31.17.212 | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:03:15.650529 |
| ip-172-31-2-159.local                | 172.31.2.159  | 64512    | BGP      | internal  | Established | in sync         | 0          | n/a                         |
| ip-172-31-21-78.local                | 172.31.21.78  | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 05:58:15.068791 |
| ip-172-31-22-95.local                | 172.31.22.95  | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 05:59:43.238465 |
| ip-172-31-23-207.local               | 172.31.23.207 | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:02:24.922901 |
| ip-172-31-25-214.local               | 172.31.25.214 | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:04:52.624323 |
| ip-172-31-30-137.local               | 172.31.30.137 | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:05:33.020029 |
| ip-172-31-4-76.local                 | 172.31.4.76   | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:12:04.853319 |
| ip-172-31-7-239.local                | 172.31.7.239  | 64512    | BGP      | internal  | Established | in sync         | 0          | n/a                         |
| ip-172-31-9-245.local                | 172.31.9.245  | 64512    | BGP      | internal  | Active      | not advertising | 1          | 2019-Jun-29 06:07:01.750834 |
[root@ip-172-31-21-119 ~]# ./contrail-introspect-cli/ist.py ctr nei | grep -w XMPP | wc -l
708
[root@ip-172-31-21-119 ~]# 

top - 06:19:56 up  1:10,  1 user,  load average: 2.04, 2.47, 2.27
Tasks: 156 total,   2 running, 154 sleeping,   0 stopped,   0 zombie
%Cpu(s): 11.5 us,  9.7 sy,  0.0 ni, 78.4 id,  0.0 wa,  0.0 hi,  0.3 si,  0.2 st
KiB Mem : 15233672 total,  7878520 free,  3006892 used,  4348260 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 11648264 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                
32368 root      20   0  890920 145632  11008 S  15.6  1.0   3:25.34 contrail-collec                                                                                        
28773 root      20   0 1357728 594448  14592 S  13.0  3.9   9:00.69 contrail-contro                                                                                        
18686 root      20   0  249228  41000   5328 R  10.3  0.3   1:00.89 python                                                                                                 
15344 root      20   0  972324  97248  35360 S   9.0  0.6   3:26.60 dockerd                                                                                                
17129 polkitd   20   0  107624  73908   1644 S   8.3  0.5   1:50.81 redis-server                                                                                           
21458 root      20   0  248352  40084   5328 S   2.7  0.3   0:41.11 python                                                                                                 
18302 root      20   0    9048   3476   2852 S   2.0  0.0   0:05.32 docker-containe                                                                                        
28757 root      20   0  248476  40196   5328 S   1.7  0.3   0:37.21 python                                                                                                 
32438 root      20   0  248496  40348   5328 S   1.7  0.3   0:34.26 python                                                                                                 
15351 root      20   0 1477100  33204  12532 S   1.3  0.2   0:16.82 docker-containe                                                                                        
18346 polkitd   20   0 2991576 563864  25552 S   1.0  3.7   5:45.65 java                                                                                                   
19994 polkitd   20   0 3880472 129392   3644 S   0.7  0.8   1:51.54 beam.smp                                                                                               
28744 root      20   0 1373980 136520  12180 S   0.7  0.9   3:13.94 contrail-dns
```

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
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/cni-tungsten-fabric.yaml.orig
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/cni-tungsten-fabric.yaml

Then you finally have kubernetes HA environment with TungstenFabric CNI, which is (mostly) up.

Note: Coredns is not active in this output, I'll fix this later in this section.

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
 - Output is the same, but it now uses MPLS encapsulation between two vRouters!

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

Note: To make coredns active, I need to make two changes
```
[root@ip-172-31-8-73 ~]# kubectl edit configmap -n kube-system coredns
-        forward . /etc/resolv.conf
+        forward . 10.47.255.253

# kubectl edit deployment -n kube-system coredns
 -> delete livenessProbe, readinessProbe
```

Then finally, coredns also is active and cluster is fully UP!

```
[root@ip-172-31-13-9 ~]# kubectl get pod --all-namespaces
NAMESPACE     NAME                                                                      READY   STATUS    RESTARTS   AGE
default       cirros-deployment-86885fbf85-pkzqz                                        1/1     Running   0          47m
default       cirros-deployment-86885fbf85-w4w6h                                        1/1     Running   0          47m
kube-system   config-zookeeper-l8m9l                                                    1/1     Running   0          24m
kube-system   config-zookeeper-lvtmq                                                    1/1     Running   0          24m
kube-system   config-zookeeper-mzlgm                                                    1/1     Running   0          24m
kube-system   contrail-agent-jc4x2                                                      2/2     Running   0          24m
kube-system   contrail-agent-psk2v                                                      2/2     Running   0          24m
kube-system   contrail-analytics-hsm7w                                                  3/3     Running   0          24m
kube-system   contrail-analytics-vgwcb                                                  3/3     Running   0          24m
kube-system   contrail-analytics-xbpwf                                                  3/3     Running   0          24m
kube-system   contrail-config-database-nodemgr-7xvnb                                    1/1     Running   0          24m
kube-system   contrail-config-database-nodemgr-9bznv                                    1/1     Running   0          24m
kube-system   contrail-config-database-nodemgr-lqtkq                                    1/1     Running   0          24m
kube-system   contrail-configdb-4svwg                                                   1/1     Running   0          24m
kube-system   contrail-configdb-gdvmc                                                   1/1     Running   0          24m
kube-system   contrail-configdb-sll25                                                   1/1     Running   0          24m
kube-system   contrail-controller-config-gmkpr                                          5/5     Running   0          24m
kube-system   contrail-controller-config-q6rvx                                          5/5     Running   0          24m
kube-system   contrail-controller-config-zbpjm                                          5/5     Running   0          24m
kube-system   contrail-controller-control-4m9fd                                         4/4     Running   0          24m
kube-system   contrail-controller-control-9klxh                                         4/4     Running   0          24m
kube-system   contrail-controller-control-wk6jp                                         4/4     Running   0          24m
kube-system   contrail-controller-webui-268bc                                           2/2     Running   0          24m
kube-system   contrail-controller-webui-57dbf                                           2/2     Running   0          24m
kube-system   contrail-controller-webui-z6c68                                           2/2     Running   0          24m
kube-system   contrail-kube-manager-6nh9d                                               1/1     Running   0          24m
kube-system   contrail-kube-manager-stqf5                                               1/1     Running   0          24m
kube-system   contrail-kube-manager-wqgl4                                               1/1     Running   0          24m
kube-system   coredns-7f865bd4f9-g8j8f                                                  1/1     Running   0          13s
kube-system   coredns-7f865bd4f9-zftsc                                                  1/1     Running   0          13s
kube-system   etcd-ip-172-31-13-9.ap-northeast-1.compute.internal                       1/1     Running   0          82m
kube-system   etcd-ip-172-31-32-58.ap-northeast-1.compute.internal                      1/1     Running   0          81m
kube-system   etcd-ip-172-31-8-73.ap-northeast-1.compute.internal                       1/1     Running   0          79m
kube-system   kube-apiserver-ip-172-31-13-9.ap-northeast-1.compute.internal             1/1     Running   0          82m
kube-system   kube-apiserver-ip-172-31-32-58.ap-northeast-1.compute.internal            1/1     Running   1          81m
kube-system   kube-apiserver-ip-172-31-8-73.ap-northeast-1.compute.internal             1/1     Running   1          80m
kube-system   kube-controller-manager-ip-172-31-13-9.ap-northeast-1.compute.internal    1/1     Running   1          83m
kube-system   kube-controller-manager-ip-172-31-32-58.ap-northeast-1.compute.internal   1/1     Running   0          80m
kube-system   kube-controller-manager-ip-172-31-8-73.ap-northeast-1.compute.internal    1/1     Running   0          80m
kube-system   kube-proxy-6ls9w                                                          1/1     Running   0          81m
kube-system   kube-proxy-82jl8                                                          1/1     Running   0          80m
kube-system   kube-proxy-bjdj9                                                          1/1     Running   0          81m
kube-system   kube-proxy-nd7hq                                                          1/1     Running   0          80m
kube-system   kube-proxy-rb7nk                                                          1/1     Running   0          83m
kube-system   kube-scheduler-ip-172-31-13-9.ap-northeast-1.compute.internal             1/1     Running   1          83m
kube-system   kube-scheduler-ip-172-31-32-58.ap-northeast-1.compute.internal            1/1     Running   0          80m
kube-system   kube-scheduler-ip-172-31-8-73.ap-northeast-1.compute.internal             1/1     Running   0          80m
kube-system   rabbitmq-b6rpx                                                            1/1     Running   0          24m
kube-system   rabbitmq-gn67t                                                            1/1     Running   0          24m
kube-system   rabbitmq-r8dvb                                                            1/1     Running   0          24m
kube-system   redis-5qvbv                                                               1/1     Running   0          24m
kube-system   redis-8mck5                                                               1/1     Running   0          24m
kube-system   redis-9d9dv                                                               1/1     Running   0          24m
[root@ip-172-31-13-9 ~]# 

[root@ip-172-31-13-9 ~]# kubectl get deployment -n kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           98m
[root@ip-172-31-13-9 ~]# 


[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py ctr route summary
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| name                                               | prefixes | paths | primary_paths | secondary_paths | infeasible_paths |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
| default-domain:default-                            | 0        | 0     | 0             | 0               | 0                |
| project:__link_local__:__link_local__.inet.0       |          |       |               |                 |                  |
| default-domain:default-project:dci-                | 0        | 0     | 0             | 0               | 0                |
| network:__default__.inet.0                         |          |       |               |                 |                  |
| default-domain:default-project:dci-network:dci-    | 3        | 8     | 0             | 8               | 0                |
| network.inet.0                                     |          |       |               |                 |                  |
| default-domain:default-project:default-virtual-    | 0        | 0     | 0             | 0               | 0                |
| network:default-virtual-network.inet.0             |          |       |               |                 |                  |
| inet.0                                             | 0        | 0     | 0             | 0               | 0                |
| default-domain:default-project:ip-fabric:ip-       | 5        | 12    | 2             | 10              | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 5        | 14    | 4             | 10              | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 5        | 12    | 2             | 10              | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]#
```

Since MP-BGP supports stitching between two clusters, those clusters are easily extended to multi-cluster environment.
 - Prefixes from each cluster will be leaked to other cluster

I'll describe the detail of this setup in Appendix section.


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
easy_install pip
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

## Container tag to be used

Container registry docker.io/opencontrailnightly has various tags.
 - https://hub.docker.com/r/opencontrailnightly/contrail-nodemgr/tags/

Let me describe some thought about what tag to be chosen in new installation.

There are three tags which I will often use, latest, 5.1.0-latest, 5.0-latest.
They are at the head of each Tungsten Fabric branch, master / R5.1 / R5.0 and various bug fix is included in each branch.
So you can choose one of those tags, for your usecase. If you need new features in R5.1, such as optional analytics components, you could choose 5.1.0-latest tag.

Since latest is the truly development branch, I don't recommend them for general use, since in some case, this build is broken to add the new features.

Other release branches are much more stable, since in most cases, they are bugfix only, although some specific period after new branch is created, release branches also seem to receive new feature.

To specifiy tags, you can use this parameter, and when git clone is typed against ansible-deployer, and contrail-container-builder, the same branch also need to be specified
```
(ansible-deployer)
git clone -b R5.1 http://github.com/Juniper/contrail-ansible-deployer

contrail_configuration:
 CONTRAIL_CONTAINER_TAG: 5.1.0-latest

(kubeadm)
git clone -b R5.1 https://github.com/Juniper/contrail-container-builder.git

common.env in contrail-container-builder repo
 CONTRAIL_CONTAINER_TAG: 5.1.0-latest
```

One point to be careful about is, since containers used with openstack (such as nova-init, neutron-init, heat-init, ...) have version dependency with openstack release, so tags might need to be changed to 5.1.0-latest-queens, 5.1.0-latest-rocky etc.

They have some openstack modules with specific version installed, so if the tags are different, openstack containers won't work well.

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

Since Tungsten Fabric have several system logs (file or docker stdout), it can easily be collected by fluentd.
 - https://www.fluentd.org/guides/recipes/docker-logging

It can be useful for administration purpose, if number of nodes are fairly large.

One more interesting subject is vRouter also supports flow exports, in a sense similar to ipfix or stateful firewall's allow / deny log .
 - https://github.com/Juniper/contrail-controller/wiki/Flow-Sampling
 - https://github.com/Juniper/contrail-specs/blob/master/security_logging_object.md

To enable this, you can set flow-export-rate > 0 (say 100), at Configure > Global Config > edit > Flow Export Rate in the Tungsten Fabric webui.

By default, it will be sent to analytics to be queried from Tungsten Fabric webui or commands like contrail-flows, contrail-sessions, but it also can be directly exported to local file to be sent to other log collectors, such as EFK, for later use.

To enable local flow logging, these parameters can be used.
```
(ansible-deployer)
contrail_configuration:
  SLO_DESTINATION: file
  SAMPLE_DESTINATION: file
(kubeadm)
env:
  SLO_DESTINATION: file
  SAMPLE_DESTINATION: file
```

If this parameters are set, log_file of vrouter-agent (such as /var/log/contrail/contrail-vrouter-agent.log) will have log output like this.

```
INFO -  [SYS_INFO]: SessionData: [ vmi = default-domain:k8s-kube-system:coredns-7f865bd4f9-6gq52__8d7eb81a-6b16-11e9-b466-0e4d26f73e5e vn = default-domain:k8s-default:k8s-default-pod-network application = application=k8s ] security_policy_rule = 00000000-0000-0000-0000-000000000004 remote_vn = default-domain:default-project:ip-fabric is_client_session = 1 is_si = 0 vrouter_ip = 172.31.6.218 local_ip = 10.47.255.250 service_port = 443 protocol = 6 sampled_forward_bytes = 132 sampled_forward_pkts = 2 sampled_reverse_bytes = 1140 sampled_reverse_pkts = 2 ip = 10.96.0.1 port = 35420 forward_flow_info= [ sampled_bytes = 132 sampled_pkts = 2 flow_uuid = 130bbb8d-9be7-46bc-8b3a-938a5c2c36bb tcp_flags = 120 setup_time = 1556609440638565 action = pass sg_rule_uuid = 00000000-0000-0000-0000-000000000004 nw_ace_uuid = 00000000-0000-0000-0000-000000000004 underlay_source_port = 55670 ] reverse_flow_info= [  sampled_bytes = 1140 sampled_pkts = 2 flow_uuid = e09e2579-7764-472b-8a35-b300c5ec34e7 tcp_flags = 120 setup_time = 1556609440638565 action = pass sg_rule_uuid = 00000000-0000-0000-0000-000000000004 nw_ace_uuid = 00000000-0000-0000-0000-000000000004 underlay_source_port = 49942 ] vm = coredns-7f865bd4f9-6gq52__8d536915-6b16-11e9-9f48-0e4d26f73e5e other_vrouter_ip = 172.31.6.218 underlay_proto = 0  ]
```

With these parameters jsonized by fluentd, and queried by kibana through ES, it is much easier to see what kind of packets went through between vRouters, or physical switches in turn.

## ElastiFlow

to be investigated

## Topology view

In my understanding, stats, logs, and topologies are three different components for software monitoring.

Promtheus and EFK part already covered stats and logs management, but since for topology there is no universal tool AFAIK, let me describe topology visualization feature in Tungsten Fabric webui.

There are a lot of topology between IT systems, but for Tungsten Fabric, those two, overlay topology and underlay topology, are most important.
 1. overlay:
  - which VNF is connected to which VN through service-chain
  - which physical device has which VN extended
 2. underlay:
  - which VM is on which vRouter
  - vRouter to leaf switch connection
  - leaf switch to spine switch connection

For overlay visualization, Monitor > Networking > Networks have some detailed view about which VNF is connected to VN through service-chain.
 - AFAIK, there is no way to see which physical device has some VN extended

For underlay visualization, Tungsten Fabric has a feature to collect lldp info through SNMP, and depict a view between leafs, spines, and vRouters and VMs.
 - http://www.opencontrail.org/wp-content/uploads/2014/11/overlay_to_physical_blogpost_image2.png
 - http://www.opencontrail.org/wp-content/uploads/2014/11/overlay_to_physical_blogpost_image3.png
 - vRouter to leaf switch connection also seems to be visualised based on arp table in leaf switch

To enable this feature in ansible-deployer installation, this role need to be added.
```
roles:
  analytics_snmp
```

Adding this, two containers, snmp-collector and topology are added to analytics nodes, and http://(analytics-ip):8081/analytics/uves/prouter/\* will be filled by PRouterLinkEntry parameter, to describe interface_name detected by lldp.
 - http://www.opencontrail.org/wp-content/uploads/2014/11/overlay_to_physical_blogpost_image1.png
 - To enable this feature, Configure > Physical Device > (edit) > 'SNMP Enabled' needs to be checked

# Day 2 operation

After installation of Tungsten Fabric is completed, users need to see the operational state such as route table and vif status, and configure various objects in Tungsten Fabric DB, such as virtual-network, logical-router, bgp-router, ...

Although Tungsten Fabric has integration with openstack neutron and kubernetes YAML to configure some parameters,
there are many situations those DBs need to be directly edited by Tungsten Fabric API or Tungsten Fabric webui.

Let me describe various options to achieve this.

## ist.py

Since ist.py is already used a lot of times in this document, there won't be much more to comment about this.
 - https://github.com/vcheny/contrail-introspect-cli

It can dump similar information with router's operational commands, based on introspect API of various Tungsten Fabric components, including route table, bgp status, components status, ...

One thing to be added is on vRouter, there are several other commands which will show similar information, such as vif, flow, vxlan, nh, rt, ...
 - https://github.com/Juniper/contrail-vrouter/tree/master/utils

Since ist.py will pick the info from vrouter-agent, and those tool pick the info from netlink, those info (mostly) always synced.
 - Still, realtime info such as vif --list --rate, flow -s will be a good addition when vRouter throughput is the key

## contrail-api-cli

When configuration update of Tungsten Fabric from CLI is needed, perhaps to use this tool will be one of the best approach.
 - https://github.com/eonpatapon/contrail-api-cli

It also can dump and traverse the contents of Tungsten Fabric DB in an intuitive way, just like Unix shell, and do ls, cat, edit and check refs and back_refs if needed.

Let me describe some commands I think it is useful.

### Installation procedure

Please type these commands to install this tool on Centos7.
```
yum -y install gcc python-devel
pip install contrail-api-cli
```

If some dependency error is shown, virtualenv might help.
```
yum -y install gcc python-devel
pip install virtualenv
virtualenv venv
source venv/bin/activate
  pip install contrail-api-cli
```

After the installation, try these commands to test TungstenFabric access. (kubernetes installation is used in this example)
```
contrail-api-cli --host xx.xx.xx.xx ls  ## xx.xx.xx.xx indicates config-api's ip
```

### ls

If you installed this tool, I firstly recommend to type this command.
```
contrail-api-cli --host xx.xx.xx.xx ls -l \*
```

Then it will just dump all the uuids inside Tungsten Fabric DB with their names!

Combining this with the cat command, a command which dumps all the configuration inside that DB can be written in a few lines, and it is highly useful to investigate what's configured.
```
for i in $(contrail-api-cli --host xx.xx.xx.xx ls \*)
do
 echo $i
 contrail-api-cli --host xx.xx.xx.xx cat $i
done
```

### cat

This command is similar to Unix cat. It dumps the json file inside Tungsten Fabric DB. To see what's configured in each element, this command can be used.
```
contrail-api-cli --host xx.xx.xx.xx ls -l virtual-network
contrail-api-cli --host xx.xx.xx.xx cat virtual-network/xxxx-xxxx-xxxx-xxxx
```

### tree

This command has two options and I think both options are useful.

This command basically dumps the refs and back_refs which one element has.

So, for example, If you want to see all the ports inside a virtual-network, this is the command you need.
```
(forward_refs)
contrail-api-cli --host xx.xx.xx.xx tree virtual-network/xxxx-xxxx-xxxx-xxxx
(back_refs)
contrail-api-cli -r --host xx.xx.xx.xx tree virtual-network/xxxx-xxxx-xxxx-xxxx
```

One additional option is -P, which dumps parent of one element. This option also would be useful in some situation.
```
contrail-api-cli -P --host xx.xx.xx.xx tree virtual-network/xxxx-xxxx-xxxx-xxxx
```


### edit

Basic idea of this command is firstly to GET a json file with a specific uuid, and save that in a temporary file, and edit this file, and PUT that with the same uuid to update the contents.
 - Similar behavior with visudo, for example

Additionally, this command could be a bit more powerful, since it supports EDITOR enviroment variable.

By default, EDITOR is defined as 'vim', but since it can be any commands or scripts, such as python files, it can be a good base of Tungsten Fabric automaion based on its REST API.
 - Since, unfortuately, no major automation tool, such as ansible, manageiq, terraform, currently supports Tungsten Fabric API directly, this might be the only way to configure Tungsten Fabric specific options, such as route-target for virtual-networks or packet-mode for ports.
 - If neutron-plugin is installed, you can also use tools like ansible, manageiq, terraform through neutron API

Basic usage of this command will be like this, to update some elements specified by a uuid.
```
contrail-api-cli edit --host xx.xx.xx.xx cat virtual-network/xxxx-xxxx-xxxx-xxxx
EDITOR=/bin/vi contrail-api-cli edit --host xx.xx.xx.xx cat virtual-network/xxxx-xxxx-xxxx-xxxx
```

If automation is an intended use case, commands similar to this could be used.
```
EDITOR=(path-of-a-script) contrail-api-cli edit --host xx.xx.xx.xx cat virtual-network/xxxx-xxxx-xxxx-xxxx


(venv) [root@ip-172-31-11-240 ~]# EDITOR=/tmp/configure-vn.py contrail-api-cli --host 172.31.11.240 edit virtual-network/035a1e3d-966b-45fd-941c-b845fd48d0c5
 -> json in Tungsten Fabric DB is updated

(venv) [root@ip-172-31-11-240 ~]# cat /tmp/configure-vn.py 
#!/usr/bin/python
import sys
import json
filename=sys.argv[1]

with open (filename) as f:
 js=json.load(f)

##print (js)
js["flood_unknown_unicast"]=True ### edit json data here

with open (filename, 'w') as f:
 json.dump(js, f)
(venv) [root@ip-172-31-11-240 ~]#
```

So with this command, you can edit Tungsten Fabric json programmatically, without deep knowledge on Tungsten Fabric API.
Since many objects are created by such as neutron API, it might be the way to use them firstly, and update them with Tungsten Fabric specific parameters (route-target is one example) by this tool.

## webui

Although there are several good CLI tools available today, historically, most of all the operation is done through Tungsten Fabric webui.

It is served at https://(controller-ip):8143, and admin / contrail123 will be the default user / pass.
 - user / pass can be changed by webui config parameter: https://github.com/Juniper/contrail-container-builder/blob/master/containers/controller/webui/base/entrypoint.sh#L248

At the top left, there are four icons, which indicates 'Monitor', 'Configure', 'Inspect', 'Query'.

Each module has those features.
 1. Monitor: This module primarily shows the status of each components, based on information from introspects, analytics UVEs, and configuration DB in some cases. (There might be some feature which won't work well if analyticsdb is not installed)
 2. Configure: Most of the configuration tasks will be done in this module.
 3. Inspect: This module has three tabs: list-of-uuid, introspect, config editor. Introspect shows the same information with ist.py. List-of-uuid, config editor will show the similar information with contrail-api-cli ls and contrail-api-cli cat / edit.
 4. Query: This module will query analyticsdb's contents. It shows the same information with the commands such as contrail-logs, contrail-flows, contrail-sessions, ... (https://github.com/Juniper/contrail-controller/wiki/Contrail-utility-scripts-for-getting-logs,-stats-and-flows-from-analytics) If analyticsdb is not installed, this module will be greyed out.

Although this webui is highly useful to grasp the current state of Tungsten Fabric, its response could be a bit slow if number of nodes are large (such as over 2,000). In that case, CLI based approach will be a bit more relevant.

## backup and restore

Backup / restore is the vital feature for important data, like SDN configuration.

Tungsten Fabric supports backup and restore through db_json_exim.py script. The procedure is described there.
 - https://www.juniper.net/documentation/en_US/contrail5.1/topics/concept/backup-using-json-50.html

Note: This repo also might be useful (restore is tested)
 - https://github.com/konono/contrail_backup

## Changing container parameters

After R5.0 and later, Tungsten Fabric components are distributed through docker containers.
Since those containers have various environment variables to change the behavior, it is sometimes necessary to update containers' environment varialbles after the installation. Let me describe how to change them.

### list of container parameters

Container parameters are used mostly in /entrypoint.sh to create conf file, which change the behavior of each microservices. To see the container environments and related parameters, it is most straightforward to see this repo.
 - https://github.com/Juniper/contrail-container-builder/tree/master/containers

This repo contains Dockerfiles and entrypoint.sh of various containers, so looking through this, you can check how to modify the parameter you need.

As an example, if you want to change the gateway parameter of vrouter-agent, you can check this file, and VROUTER_GATEWAY is directly used to replace that parameter.
 - https://github.com/Juniper/contrail-container-builder/blob/master/containers/vrouter/agent/entrypoint.sh

```
[VIRTUAL-HOST-INTERFACE]
name=vhost0
ip=$vrouter_cidr
physical_interface=$phys_int
gateway=$VROUTER_GATEWAY   ### this is the container environment variable which needs to be changed
compute_node_address=$vrouter_ip
```

So if you know the parameter of microservice you need, you can check the corresponding container environment variable.

Note that in some cases, there is no container environment variable which directly modify the microservices parameters.

In that case, you can use add_ini_params_from_env function, which is at the last part of each entrypoint.sh.
```
add_ini_params_from_env VROUTER_AGENT /etc/contrail/contrail-vrouter-agent.conf
```

In that case, if you give this environment variable,
```
VROUTER_AGENT__FLOWS__thread_count=8
```
it can be translated to [FLOWS], thread_count=8, so with that method, you can directly modify microservices' conf file, even if no handy parameters are supplied to modify this.

### ansible-deployer

If ansible-deployer is used, it uses docker-compose to create docker containers, and environment variables are defined in /etc/contrail/common_xxx.env. (xxx is the rolename)

So if you want to update such as vrouter parameters, you can edit /etc/contrail/common_vrouter.env, and type these commands.
```
docker-compose -f /etc/contrail/vrouter/docker-compose.yaml down
docker-compose -f /etc/contrail/vrouter/docker-compose.yaml up -d
```

Then vrouter containers are recreated and the new parameters are applied.

### kubeadm

If kubeadm and kubernetes yaml is used to install Tungsten Fabric containers, each container uses configmap named 'env' as the source of environment variables.
So you can type this command to edit environment variables, and can delete some Tungsten Fabric pods to recreate the containers. (Since containers are defined as DaemonMap, it will be recreated automatically)
```
kubectl edit configmap -n kube-system env
```

# Troubleshooting Tips

When using vRouter, there could be some situations routing won't work well as expected.

I have gathered most common issues and steps to investigate vRouter's routing behavior.

To investigate this, there are two ways, one is to see config detail related, and the other is to see operational state of control and vRouter.

Since for the former, contrail-api-cli is most useful, and the latter, ist.py is (especailly for remote-debug),
let me describe some info in this format.

Note: if these tools are not available, you can use curl for that purpose.

For example, when
```
source /etc/kolla/kolla_toolbox/admin-openrc.sh
contrail-api-cli --host x.x.x.x ls -l virtual-network
contrail-api-cli --host x.x.x.x cat virtual-network/xxxx-xxxx-xxxx-xxxx
```
is needed, those command collect the same info.
```
source /etc/kolla/kolla_toolbox/admin-openrc.sh
openstack token issue
curl -H 'x-auth-token: tokenid' x.x.x.x:8082/virtual-networks
curl -H 'x-auth-token: tokenid' x.x.x.x:8082/virtual-network/xxxx-xxxx-xxxx-xxxx
```

In a similar manner, ist.py also can be replaced by various curl command.
Let me describe curl command for most common case. (cli is more memorable though)
```
ist.py ctr route show
   curl control-ip:8083/Snh_ShowRouteReq
ist.py ctr route nei
   curl control-ip:8083/Snh_BgpNeighborReq
ist.py vr intf
   curl vrouter-ip:8085/Snh_ItfReq
ist.py vr vrf
   curl vrouter-ip:8085/Snh_VrfListReq
ist.py vr route -v vrf-id
   curl vrouter-ip:8085/Snh_Inet4UcRouteReq?vrf_index=vrf-id
```

Additionally, ifmap information (it is comparable to device configuration for vRouter, such as interface, vrf, virtual-machine, ...) is also useful to see what is configured.

It can be seen by those commands.
```
ist.py ctr ifmap table
   curl control-ip:8083/Snh_IFMapNodeTableListShowReq
ist.py ctr ifmap table virtual-network
   curl control-ip:8083/Snh_IFMapTableShowReq?table_name=virtual-network

ist.py ctr ifmap client
   curl control-ip:8083/Snh_IFMapPerClientLinksShowReq
ist.py ctr ifmap node
   curl control-ip:8083/Snh_IFMapLinkTableShowReq
ist.py ctr ifmap link
   curl control-ip:8083/Snh_IFMapNodeShowReq

ist.py vr ifmap
   curl vrouter-ip:8085/Snh_ShowIFMapAgentReq
```

### x. some VM-to-VM packets can't reach the other node
 To investigate this, firstly, it needs to be seen that is control plane issue or data plane issue.
 For control plane issue, those commands will be most useful.
```
 # ist.py ctr route show
 # ist.py vr intf
 # ist.py vr vrf
 # ist.py vr route -v (vrf id)
```

 If routing seems ok, you can firstly see if packet is arrived at the destination vrouter by tcpdump.
```
 # tcpdump -i any -nn udp port 6635 or udp port 4789 or proto gre or icmp # for physical NIC
 # tcpdump -i any -nn icmp # for tap device
```

 When packet reached the destination vRouter, check 
```
 # flow -l
```
 to see if it is dropped by flow action.
  - For example, action: D(Policy), D(SG) indicates it is dropped by network-policy or security-group
 To investigate the flow action further, those command would help.
```
 # ist.py vr intf -f text
 # ist.py vr acl
```

Note: To see the reason of packet drop, dropstats command could have some more info. 
```
 # watch -n 1 'dropstats | grep -v -w 0'
 # watch -n 1 'vif --get 0 --get-drop-stats'
 # watch -n 1 'vif --get n --get-drop-stats' (n is vif id)
 # ping -i 0.2 overlay-ip # this can be used to see specific dropstats counter is incrementing because of that packets
```

### x. I can't access VMs from external nodes
 check
```
 # flow -l
```
to see flow action of this packet. If action was D(SG), it is dropped by security-group, so it need to be changed to permit external access (default for openstack ingress rule is to allow VM-to-VM access only)

### x. kubernetes service / ingress won't up, SNAT with floating-ip won't work well
 Since those are set up by svc-monitor, you can firstly check
```
 # tail -f /var/log/contrail/contrail-svc-monitor.log
```
 to see some error is seen.
  - One example is 'No vRouter is availale' is logged, so they can't start those service. That is caused by NodeStatus from vRouter to analytics-api is 'Non-Functional' for some reason, so it needs to be investigated from vRouter side.

 If svc-monitor is working well, you need to investigate behavior of load-balancer object.

 When service is used, it adds ecmp route to reach application, so those commands can be used to investigate control plane (same procedure to see VM-to-VM routing)
```
 # ist.py ctr route show
 # ist.py vr route -v (vrf-id)
```

 When ingress or SNAT is used, it will start haproxy process inside linux namespace in vRouter container.
 To investigate the detail, you can try those commands, to see those name
```
 # docker exec -it vrouter-agent bash
   # ip netns
   # ip netns exec vrouter-xxx ip -o a
   # ip netns exec vrouter-xxx ip route
   # ip netns exec vrouter-xxx iptables -L -n -v -t nat
 # tail -f /var/log/messages # haproxy log is logged
```

 Since ingress service and SNAT also use vRouter routing, those commands also are helpful to see prefixes to those service are exported to vRouter's routing table.
```
 # ist.py ctr route show
 # ist.py vr vrf
 # ist.py vr route -v (vrf-id)
```

### x. service-chain won't work well
 Since service-chain use change vRouter routing table, firstly those commands can be used, to see if routing-instances are successfully created, and ServiceChain route is correctly imported
```
 # ist.py ctr route summary
 # ist.py ctr route show
 # ist.py ctr route show -p ServiceChain
 # ist.py ctr sc
```

 If control plane works well, you need to investigate data plane behaivor in the same manner with VM-to-VM traffic (security-group also can block service-chain traffic, so please also check that to investigate service-chain from external traffic)
```
 # tcpdump -i any -nn udp port 6635 or udp port 4789 or proto gre or icmp
 # ist.py vr intf
 # ist.py vr vrf
 # ist.py vr route -v (vrf-id)
 # flow -l
```

### x. distributed SNAT won't work well
 That feature is implemented by ACL on vRouter, so to investigate this feature, this command is useful.
```
 # ist.py vr intf -f text
```
 If icmp works well and tcp / udp won't work well, please also check port lists are specified.

### x. cni returned Poll VM-CFG 404 error

In kubernetes deployment, cni sometimes returns this error and won't assign IPs to pod. (This is seen in various place such kubectl describe pod)
```
networkPlugin cni failed to set up pod "coredns-5644d7b6d9-p8fkk_kube-system" network: Failed in Poll VM-CFG. Error : Failed in PollVM. Error : Failed HTTP Get operation. Return code 404
```

This message is a generic error, and caused by several reasons .. 

Internally, when pod is created, cni tries to receive its IP from vrouter-agent, which in turn receive that from control process by XMPP. 
 - that is based on virtual-machine-interface info, which is created by kube-manager from kube-apiserver info.

So to fix this issue, serveral steps need to be done.

1. contrail-status on controller node
 - config-api, control needs to be in ‘active’ state

2. contrail-status on contrail-kube-manager node is in 'active' state
 - this process will retrieve the info from kube-apiserver and create pod / load balancer etc on config-api
 
3. contrail-status on vrouter node
 - vrouter-agent needs to be in ‘active’ state
 
4. if standalone kubernetes yaml is used, it has known limitation about race condition between vrouter registration and vrouter-agent restart.
Restarting control might resolve this issue.
```
 # docker restart control_control_1
```
 
5. if everything is fine,
 - /var/log/contrail/contrail-kube-manager.log
 - /var/log/contrail/api-zk.log
 - /var/log/contrail/contrail-vrouter-agent.log
 - /var/log/contrail/cni/opencontrail.log <- cni log

needs to be investigated further ..
 - root cause might be xmpp issue, underlay issue, /etc/hosts issue and so on

### x. disk is full

When disk size is, say, 50GB, it might be full after one week or so from installtion.
When this occured, analytics data need to be removed and analytics database need to be restarted.
```
[check analytics db size]
du -smx /var/lib/docker/volumes/analytics_database_analytics_cassandra/_data/ContrailAnalyticsCql

[if it is large, remove by this]
rm -rf /var/lib/docker/volumes/analytics_database_analytics_cassandra/_data/ContrailAnalyticsCql
docker-compose -f /etc/contrail/analytics_database/docker-compose.yaml down
docker-compose -f /etc/contrail/analytics_database/docker-compose.yaml up -d
```

To avoid this issue in future, this knob can be used.
```
echo 'ANALYTICS_STATISTICS_TTL=2' >> /etc/contrail/common_analytics.env
docker-compose -f /etc/contrail/analytics/docker-compose.yaml down
docker-compose -f /etc/contrail/analytics/docker-compose.yaml up -d
```

### x. analytics cassandra state detected down

If this error is seen in contrail-status, it states analytics cassandra is not working well.
```
== Contrail database ==
nodemgr: initializing (Cassandra state detected DOWN.)
```

If JVM_EXTRA_OPTS: "-Xms128m -Xmx1g" is set, most likely the cause is java's OutOfMemory error, so it can be updated to something like
```
JVM_EXTRA_OPTS: "-Xms128m -Xmx2g"
```
in /etc/contrail/common.env,
and analytics database can be restarted.
```
docker-compose -f /etc/contrail/analytics_database/docker-compose.yaml down
docker-compose -f /etc/contrail/analytics_database/docker-compose.yaml up -d
```


# Appendix

## Cluster update

Cluster-wide update is an important subject, to keep SLA of the production cluster, with the newest features still avaiable in that cluster.

Since Tungsten Fabric uses similar protocol with MPLS-VPN, even if the module version of control and vRouter is different, basic interoperability is available, as far as I tried.

So the general idea is, firstly update controllers one by one, and update vRouters, one by one, with vMotion or maintence mode if needed.

Let me describe this procedure firstly.

Additionally, Tungsten Fabric controller supports curious feature named ISSU, although I think this name is a bit confusing, since Tungsten Fabric controller is much more similar to route reflector, rather rhan routing-engine.
 - https://github.com/Juniper/contrail-specs/blob/master/contrail-issu.md

So basic idea is, firstly replicate all the configs to the newly created controllers (or route reflectors), and after that, update vRouter settings (and vRouter modules if servers can be rebooted) to use those new controllers. With this procedure, rollback operation of vRouter module update also will be much easier.

Let me descirbe this procedure later in this chapter.

### in-place update

Since ansible-deployer follows idempotent behavior, update is not much different from install.
These command will update all the modules.
```
cd contrail-ansible-deployer
git pull
vi config/instances.yaml
(update CONTRAIL_CONTAINER_TAG)
ansible-playbook -e orchestrator=xxx -i inventory/ playbooks/install_contrail.yml
```

One caveat is, since this command restarts all the nodes mostly simultaneously, it is not easy to restart controllers and vRouters one-by-one.
Additionally, removing other nodes from instances.yaml won't work, since one node's update require some parameters of other nodes.
 - For example, vRouter update needs controls' ip, that is deduced from control role nodes in instances.yaml

To overcome this, since ansible-deployer chooses hosts to be modified based on 
```
provider: bms
```
parameter, changing them to such as
```
provider: bms-maint
```
except the one which will be updated, will be one possible workaround.
 - As far as I tried, this procedure makes it possible to update nodes one-by-one, although I don't know it will always work well ..

### ISSU

ISSU can be used even if container formats are largely different, like 4.x to 5.x case, since it creates a new cluster of controllers, and copy the data inside.

Firstly, I'll describe the simplest case, 1 old controller and 1 new controller to see the overall procedure. All the commands are typed at the new controller.

```
old-controller:
 ip: 172.31.2.209
 hostname: ip-172-31-2-209
new-controller:
 ip: 172.31.1.154
 hostname: ip-172-31-1-154

(both controllers are installed with this instances.yaml)
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
   ip: x.x.x.x ## controller's ip
contrail_configuration:
  CONTRAIL_CONTAINER_TAG: r5.1
  KUBERNETES_CLUSTER_PROJECT: {}
  JVM_EXTRA_OPTS: "-Xms128m -Xmx1g"
global_configuration:
  CONTAINER_REGISTRY: tungstenfabric

[commands]
1. stop batch jobs
docker stop config_devicemgr_1
docker stop config_schema_1
docker stop config_svcmonitor_1

2. register new control in cassandra and set up bgp between them
docker exec -it config_api_1 bash
python /opt/contrail/utils/provision_control.py --host_name ip-172-31-1-154.local --host_ip 172.31.1.154 --api_server_ip 172.31.2.209 --api_server_port 8082 --oper add --router_asn 64512 --ibgp_auto_mesh

3. sync the data between controllers
vi contrail-issu.conf
(write down this)
[DEFAULTS]
old_rabbit_address_list = 172.31.2.209
old_rabbit_port = 5673
new_rabbit_address_list = 172.31.1.154
new_rabbit_port = 5673
old_cassandra_address_list = 172.31.2.209:9161
old_zookeeper_address_list = 172.31.2.209:2181
new_cassandra_address_list = 172.31.1.154:9161
new_zookeeper_address_list = 172.31.1.154:2181
new_api_info={"172.31.1.154": [("root"), ("password")]} ## ssh public-key can be used

image_id=`docker images | awk '/config-api/{print $3}' | head -1`

docker run --rm -it --network host -v $(pwd)/contrail-issu.conf:/etc/contrail/contrail-issu.conf --entrypoint /bin/bash -v /root/.ssh:/root/.ssh $image_id -c "/usr/bin/contrail-issu-pre-sync -c /etc/contrail/contrail-issu.conf"

4. start the process to do real-time data sync
docker run --rm --detach -it --network host -v $(pwd)/contrail-issu.conf:/etc/contrail/contrail-issu.conf --entrypoint /bin/bash -v /root/.ssh:/root/.ssh --name issu-run-sync $image_id -c "/usr/bin/contrail-issu-run-sync -c /etc/contrail/contrail-issu.conf"

(check the log if needed)
docker exec -t issu-run-sync tail -f /var/log/contrail/issu_contrail_run_sync.log

5. (update vrouters)

6. stop the job and sync all the data when finished
docker rm -f issu-run-sync

image_id=`docker images | awk '/config-api/{print $3}' | head -1`
docker run --rm -it --network host -v $(pwd)/contrail-issu.conf:/etc/contrail/contrail-issu.conf --entrypoint /bin/bash -v /root/.ssh:/root/.ssh --name issu-run-sync $image_id -c "/usr/bin/contrail-issu-post-sync -c /etc/contrail/contrail-issu.conf"
docker run --rm -it --network host -v $(pwd)/contrail-issu.conf:/etc/contrail/contrail-issu.conf --entrypoint /bin/bash -v /root/.ssh:/root/.ssh --name issu-run-sync $image_id -c "/usr/bin/contrail-issu-zk-sync -c /etc/contrail/contrail-issu.conf"

7. remove old nodes from cassandra and add new nodes
vi issu.conf
(write down this)
[DEFAULTS]
db_host_info={"172.31.1.154": "ip-172-31-1-154.local"}
config_host_info={"172.31.1.154": "ip-172-31-1-154.local"}
analytics_host_info={"172.31.1.154": "ip-172-31-1-154.local"}
control_host_info={"172.31.1.154": "ip-172-31-1-154.local"}
api_server_ip=172.31.1.154

docker cp issu.conf config_api_1:issu.conf
docker exec -it config_api_1 python /opt/contrail/utils/provision_issu.py -c issu.conf

8. start batch jobs
docker start config_devicemgr_1
docker start config_schema_1
docker start config_svcmonitor_1
```

These will be possible checkpoints.
1. After 3, you can try contrail-api-cli ls -l \\* to see all the data are copied successfully, and ist.py ctr nei to see ibgp is up between controllers.
2. After 4, old db can be modified, to see the changes are successfully propagated to new db.

After this, I will cover more realistic case with orchestrators and two vRouters.

### Orchestrator integration

To illustrate the case combined with orchestrators, I tried two vRouters and kubernetes setup with ansible-deployer.

Even when combined with orchestrators, overall procedure won't be much different.

One thing which need to be careful about is, when kube-manager needs to be changed to the new one.

Since kube-manager dynamically subscribe events from kube-apiserver and update Tungsten Fabric config-database, in a sense, it is similar to batch jobs, such as schema-transformer, svc-monitor and device-manager.
So I stopped and started old or new kube-manager (and actually webui also) at the same time with such batch jobs, but it might need to be changed based on each setup.

So overall procedure in this case will be the following.
```
1. setup one controller (with one kube-manager and kubernetes-master) and two vRouters
2. setup one new controller (with one kube-manager, but kubernetes-master is the same one with the old controller) 
3. stop batch jobs and kube-manager, webui for new controller
4. start issu procedure and continue that until starting run-sync
 -> iBGP will be established between controllers
5. update vRouters one-by-one based on new controller's ansible-deployer
  -> When one vRouter is moved to the new one, new controller also will got the route-target of k8s-default-pod-network,
   and ping between containers remain working well (ist.py ctr route summary and the ping result will be attached later)
6. When all the vRouters are moved to the new controller, stop batch jobs and kube-manager, webui on the old controller
   After that, continue issu procedure, and start batch jobs and kube-manager and webui, on the new controller
  -> From the beginning and till the end of this phase, you can't change config-database manually, so some maintenance time might be needed
  (it could last up to 5-15 min, ping is ok, but new container creation won't work well, until new kube-manager is started)
7. finally, stop control, config and config-database on the old node
```

When updating vRouters, I used provider: bms-maint for controller, k8s_master, and vRouters which are already changed to new one, to avoid disturbing based on container restart.
I'll attach original instances.yaml and instances.yaml to update vRouters, for further detail.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/instances-yaml-for-ISSU

I'll also attach the result of ist.py ctr nei and ist.py ctr route summary on each phase, to illustrate the detail of what's going on.
 - Let me note that I actually don't update the module in this example, since this setup is primarily to highlight the ISSU procedure (since ansible-deployer re-create vrouter-agent containers even when module version is the same, number of packet loss won't be much different even if the actual module update is done)

```
old-controller: 172.31.19.25
new-controller: 172.31.13.9
two-vRouters: 172.31.25.102, 172.31.33.175

Before stating issu:

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr nei
Introspect Host: 172.31.19.25
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-25-102.local | 172.31.25.102 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-33-175.local | 172.31.33.175 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]# 
[root@ip-172-31-13-9 ~]# 
[root@ip-172-31-13-9 ~]# 
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr nei
Introspect Host: 172.31.13.9
[root@ip-172-31-13-9 ~]# 
 -> iBGP is not configured yet

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr route summary
Introspect Host: 172.31.19.25
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
| default-domain:default-project:ip-fabric:ip-       | 7        | 7     | 2             | 5               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 7        | 7     | 4             | 3               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 7        | 7     | 1             | 6               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]# 
[root@ip-172-31-13-9 ~]# 
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr route summary
Introspect Host: 172.31.13.9
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
| default-domain:default-project:ip-fabric:ip-       | 0        | 0     | 0             | 0               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 0        | 0     | 0             | 0               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 0        | 0     | 0             | 0               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]# 
 -> No route is imported in the new controller


[root@ip-172-31-19-25 contrail-ansible-deployer]# kubectl get pod -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE
cirros-deployment-75c98888b9-6qmcm   1/1     Running   0          4m58s   10.47.255.249   ip-172-31-25-102.ap-northeast-1.compute.internal   <none>
cirros-deployment-75c98888b9-lxq4k   1/1     Running   0          4m58s   10.47.255.250   ip-172-31-33-175.ap-northeast-1.compute.internal   <none>
[root@ip-172-31-19-25 contrail-ansible-deployer]# 

/ # ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue \    link/ether 02:6b:dc:98:ac:95 brd ff:ff:ff:ff:ff:ff
13: eth0    inet 10.47.255.249/12 scope global eth0\       valid_lft forever preferred_lft forever
/ # ping 10.47.255.250
PING 10.47.255.250 (10.47.255.250): 56 data bytes
64 bytes from 10.47.255.250: seq=0 ttl=63 time=2.155 ms
64 bytes from 10.47.255.250: seq=1 ttl=63 time=0.904 ms
^C
--- 10.47.255.250 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.904/1.529/2.155 ms
/ # 
 -> two vRouters have one container on each node, and ping between two containers are working well


After provision_control:

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr nei
Introspect Host: 172.31.19.25
+------------------------+---------------+----------+----------+-----------+-------------+-----------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state      | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+-----------------+------------+-----------+
| ip-172-31-13-9.local   | 172.31.13.9   | 64512    | BGP      | internal  | Idle        | not advertising | 0          | n/a       |
| ip-172-31-25-102.local | 172.31.25.102 | 0        | XMPP     | internal  | Established | in sync         | 0          | n/a       |
| ip-172-31-33-175.local | 172.31.33.175 | 0        | XMPP     | internal  | Established | in sync         | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+-----------------+------------+-----------+
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr nei
Introspect Host: 172.31.13.9
[root@ip-172-31-13-9 ~]#
 -> iBGP is configured at the old controller, but new controller doesn't have that configuration yet (that will be replicated to the new controller, when pre-sync is performed)


After run-sync:
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr nei
Introspect Host: 172.31.19.25
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-13-9.local   | 172.31.13.9   | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-25-102.local | 172.31.25.102 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-33-175.local | 172.31.33.175 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr nei
Introspect Host: 172.31.13.9
+-----------------------+--------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                  | peer_address | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+-----------------------+--------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-19-25.local | 172.31.19.25 | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
+-----------------------+--------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]#
 -> iBGP is established, ctr route summary haven't yet changed, since new controller don't have k8s-default-pod-network's route-target and route target filtering prohibits importing those prefixes


After moving one node to the new controller:

/ # ping 10.47.255.250
PING 10.47.255.250 (10.47.255.250): 56 data bytes
64 bytes from 10.47.255.250: seq=0 ttl=63 time=1.684 ms
64 bytes from 10.47.255.250: seq=1 ttl=63 time=0.835 ms
64 bytes from 10.47.255.250: seq=2 ttl=63 time=0.836 ms
(snip)
64 bytes from 10.47.255.250: seq=37 ttl=63 time=0.878 ms
64 bytes from 10.47.255.250: seq=38 ttl=63 time=0.823 ms
64 bytes from 10.47.255.250: seq=39 ttl=63 time=0.820 ms
64 bytes from 10.47.255.250: seq=40 ttl=63 time=1.364 ms
64 bytes from 10.47.255.250: seq=44 ttl=63 time=2.209 ms
64 bytes from 10.47.255.250: seq=45 ttl=63 time=0.869 ms
64 bytes from 10.47.255.250: seq=46 ttl=63 time=0.857 ms
64 bytes from 10.47.255.250: seq=47 ttl=63 time=0.855 ms
64 bytes from 10.47.255.250: seq=48 ttl=63 time=0.845 ms
64 bytes from 10.47.255.250: seq=49 ttl=63 time=0.842 ms
64 bytes from 10.47.255.250: seq=50 ttl=63 time=0.885 ms
64 bytes from 10.47.255.250: seq=51 ttl=63 time=0.891 ms
64 bytes from 10.47.255.250: seq=52 ttl=63 time=0.909 ms
64 bytes from 10.47.255.250: seq=53 ttl=63 time=0.867 ms
64 bytes from 10.47.255.250: seq=54 ttl=63 time=0.884 ms
64 bytes from 10.47.255.250: seq=55 ttl=63 time=0.865 ms
64 bytes from 10.47.255.250: seq=56 ttl=63 time=0.840 ms
64 bytes from 10.47.255.250: seq=57 ttl=63 time=0.877 ms
^C
--- 10.47.255.250 ping statistics ---
58 packets transmitted, 55 packets received, 5% packet loss
round-trip min/avg/max = 0.810/0.930/2.209 ms
/ # 
 -> When vrouter-agent is restarted, 3 packet loss is seen (seq 40-44). After moving one vRouter to the new one, ping remain working well.


[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr nei
Introspect Host: 172.31.19.25
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-13-9.local   | 172.31.13.9   | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-33-175.local | 172.31.33.175 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr nei
Introspect Host: 172.31.13.9
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-19-25.local  | 172.31.19.25  | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-25-102.local | 172.31.25.102 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]# 
 -> Both controllers have one XMPP connection and Established iBGP

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr route summary
Introspect Host: 172.31.19.25
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
| default-domain:default-project:ip-fabric:ip-       | 7        | 7     | 1             | 6               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 7        | 7     | 1             | 6               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 7        | 7     | 0             | 7               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr route summary
Introspect Host: 172.31.13.9
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
| default-domain:default-project:ip-fabric:ip-       | 7        | 7     | 1             | 6               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 7        | 7     | 3             | 4               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 7        | 7     | 1             | 6               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]# 
 -> Since both controllers have at least one container from k8s-default-pod-network, they used iBGP to exchange the prefixes between them, and they have the same prefixes


After moving the second vrouter to the new controller:
/ # ping 10.47.255.250
PING 10.47.255.250 (10.47.255.250): 56 data bytes
64 bytes from 10.47.255.250: seq=0 ttl=63 time=1.750 ms
64 bytes from 10.47.255.250: seq=1 ttl=63 time=0.815 ms
64 bytes from 10.47.255.250: seq=2 ttl=63 time=0.851 ms
64 bytes from 10.47.255.250: seq=3 ttl=63 time=0.809 ms
(snip)
64 bytes from 10.47.255.250: seq=34 ttl=63 time=0.853 ms
64 bytes from 10.47.255.250: seq=35 ttl=63 time=0.848 ms
64 bytes from 10.47.255.250: seq=36 ttl=63 time=0.833 ms
64 bytes from 10.47.255.250: seq=37 ttl=63 time=0.832 ms
64 bytes from 10.47.255.250: seq=38 ttl=63 time=0.910 ms
64 bytes from 10.47.255.250: seq=42 ttl=63 time=2.071 ms
64 bytes from 10.47.255.250: seq=43 ttl=63 time=0.826 ms
64 bytes from 10.47.255.250: seq=44 ttl=63 time=0.853 ms
64 bytes from 10.47.255.250: seq=45 ttl=63 time=0.851 ms
64 bytes from 10.47.255.250: seq=46 ttl=63 time=0.853 ms
64 bytes from 10.47.255.250: seq=47 ttl=63 time=0.851 ms
64 bytes from 10.47.255.250: seq=48 ttl=63 time=0.855 ms
64 bytes from 10.47.255.250: seq=49 ttl=63 time=0.869 ms
64 bytes from 10.47.255.250: seq=50 ttl=63 time=0.833 ms
64 bytes from 10.47.255.250: seq=51 ttl=63 time=0.859 ms
64 bytes from 10.47.255.250: seq=52 ttl=63 time=0.866 ms
64 bytes from 10.47.255.250: seq=53 ttl=63 time=0.840 ms
64 bytes from 10.47.255.250: seq=54 ttl=63 time=0.841 ms
64 bytes from 10.47.255.250: seq=55 ttl=63 time=0.854 ms
^C
--- 10.47.255.250 ping statistics ---
56 packets transmitted, 53 packets received, 5% packet loss
round-trip min/avg/max = 0.799/0.888/2.071 ms
/ #
 -> 3 packet loss is seen (seq 38-42)


[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr nei
Introspect Host: 172.31.19.25
+----------------------+--------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                 | peer_address | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+----------------------+--------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-13-9.local | 172.31.13.9  | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
+----------------------+--------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr nei
Introspect Host: 172.31.13.9
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-19-25.local  | 172.31.19.25  | 64512    | BGP      | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-25-102.local | 172.31.25.102 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-33-175.local | 172.31.33.175 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]# 
 -> New controller has two XMPP connections.

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr route summary
Introspect Host: 172.31.19.25
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
| default-domain:default-project:ip-fabric:ip-       | 0        | 0     | 0             | 0               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 0        | 0     | 0             | 0               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 0        | 0     | 0             | 0               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr route summary
Introspect Host: 172.31.13.9
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
| default-domain:default-project:ip-fabric:ip-       | 7        | 7     | 2             | 5               | 0                |
| fabric.inet.0                                      |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-pod-network | 7        | 7     | 4             | 3               | 0                |
| :k8s-default-pod-network.inet.0                    |          |       |               |                 |                  |
| default-domain:k8s-default:k8s-default-service-    | 7        | 7     | 1             | 6               | 0                |
| network:k8s-default-service-network.inet.0         |          |       |               |                 |                  |
+----------------------------------------------------+----------+-------+---------------+-----------------+------------------+
[root@ip-172-31-13-9 ~]#
 -> Old controller doesn't have prefixes anymore


After ISSU procedure finished and new kube-manager started:
[root@ip-172-31-19-25 ~]# kubectl get pod -o wide
NAME                                  READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE
cirros-deployment-75c98888b9-6qmcm    1/1     Running   0          34m   10.47.255.249   ip-172-31-25-102.ap-northeast-1.compute.internal   <none>
cirros-deployment-75c98888b9-lxq4k    1/1     Running   0          34m   10.47.255.250   ip-172-31-33-175.ap-northeast-1.compute.internal   <none>
cirros-deployment2-648b98685f-b8pxw   1/1     Running   0          15s   10.47.255.247   ip-172-31-25-102.ap-northeast-1.compute.internal   <none>
cirros-deployment2-648b98685f-nv7z9   1/1     Running   0          15s   10.47.255.248   ip-172-31-33-175.ap-northeast-1.compute.internal   <none>
[root@ip-172-31-19-25 ~]# 
 -> containers can be created with new ip (10.47.255.247, 10.47.255.248 are the new ip from new controller)

[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.19.25 ctr nei
Introspect Host: 172.31.19.25
+----------------------+--------------+----------+----------+-----------+--------+-----------------+------------+-----------------------------+
| peer                 | peer_address | peer_asn | encoding | peer_type | state  | send_state      | flap_count | flap_time                   |
+----------------------+--------------+----------+----------+-----------+--------+-----------------+------------+-----------------------------+
| ip-172-31-13-9.local | 172.31.13.9  | 64512    | BGP      | internal  | Active | not advertising | 1          | 2019-Jun-23 05:37:02.614003 |
+----------------------+--------------+----------+----------+-----------+--------+-----------------+------------+-----------------------------+
[root@ip-172-31-13-9 ~]# ./contrail-introspect-cli/ist.py --host 172.31.13.9 ctr nei
Introspect Host: 172.31.13.9
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| peer                   | peer_address  | peer_asn | encoding | peer_type | state       | send_state | flap_count | flap_time |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
| ip-172-31-25-102.local | 172.31.25.102 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
| ip-172-31-33-175.local | 172.31.33.175 | 0        | XMPP     | internal  | Established | in sync    | 0          | n/a       |
+------------------------+---------------+----------+----------+-----------+-------------+------------+------------+-----------+
[root@ip-172-31-13-9 ~]#
 -> New controller doesn't have an iBGP to the old controller anymore. Old controller still has an iBGP entry, although this process will be stopped soon :)


After stopping old control, config:
[root@ip-172-31-19-25 ~]# kubectl get pod -o wide
NAME                                  READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE
cirros-deployment-75c98888b9-6qmcm    1/1     Running   0          48m   10.47.255.249   ip-172-31-25-102.ap-northeast-1.compute.internal   <none>
cirros-deployment-75c98888b9-lxq4k    1/1     Running   0          48m   10.47.255.250   ip-172-31-33-175.ap-northeast-1.compute.internal   <none>
cirros-deployment2-648b98685f-b8pxw   1/1     Running   0          13m   10.47.255.247   ip-172-31-25-102.ap-northeast-1.compute.internal   <none>
cirros-deployment2-648b98685f-nv7z9   1/1     Running   0          13m   10.47.255.248   ip-172-31-33-175.ap-northeast-1.compute.internal   <none>
cirros-deployment3-68fb484676-ct9q9   1/1     Running   0          18s   10.47.255.245   ip-172-31-25-102.ap-northeast-1.compute.internal   <none>
cirros-deployment3-68fb484676-mxbzq   1/1     Running   0          18s   10.47.255.246   ip-172-31-33-175.ap-northeast-1.compute.internal   <none>
[root@ip-172-31-19-25 ~]# 
 -> New containers still can be created

[root@ip-172-31-25-102 ~]# contrail-status 
Pod      Service  Original Name           State    Id            Status         
vrouter  agent    contrail-vrouter-agent  running  9a46a1a721a7  Up 33 minutes  
vrouter  nodemgr  contrail-nodemgr        running  11fb0a7bc86d  Up 33 minutes  

vrouter kernel module is PRESENT
== Contrail vrouter ==
nodemgr: active
agent: active

[root@ip-172-31-25-102 ~]# 
 -> vRouter is working well with the new control

/ # ping 10.47.255.250
PING 10.47.255.250 (10.47.255.250): 56 data bytes
64 bytes from 10.47.255.250: seq=0 ttl=63 time=1.781 ms
64 bytes from 10.47.255.250: seq=1 ttl=63 time=0.857 ms
^C
--- 10.47.255.250 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.857/1.319/1.781 ms
/ #
 -> ping between vRouters is ok
```

### Backward compatibility

Since there are several methods to update cluster (in-place and ISSU, ifdown vhost0 or not), which method is needed is an important subject.

Before discussing the detail of this, let me describe the behavior of vrouter-agent up / down, and ifup vhost0 / ifdown vhost0.

When vrouter-agent is updated, one assumption is that both of vrouter-agent container, and vhost0 is re-created.

Actually, this is not the case, since vhost0 is tightly coupled with vrouter.ko, and it needs to be deleted at the same time with vrouter.ko is unloaded from the kernel.
So from operational point of view, ifdown vhost0 is needed, if not only vrouter-agent but also vrouter.ko need to be updated. (ifdown vhost0 also will do rmmod vrouter internally)

So to discuss backward compatibility, there are three topics to be investigated.
 1. controller to vrouter-agent compatibility
  - if no backward compatibility is available, ISSU is needed
 2. vrouter-agent to vrouter.ko compatibility
  - if no backward compatibility is available, ifdown vhost0 is needed, which causes minimal 5-10 sec traffic loss, so it practically means traffic need to be moved to other nodes, with such as live migration
   - Since vrouter-agent uses netlink to sync data with vrouter.ko, schema change could leads to unexpected behavior of vrouter-agent (such as segmentation fault of vrouter-agent at Ksync logic)
 3. vrouter.ko to kernel compatiblity
  - if no backward compatibility is available, kernel also needs to be updated, so it means traffic need to be moved to other nodes
  - when vrouter.ko has different in-kernal API, it can't be loaded by kernel, and vhost0 and vrouter-agent can't be created

For 2 and 3, since kernel update is unavoidable for various reasons,
one possible plan is to firstly choose one new kernel version, and choose one vrouter-agent / vrouter.ko which supports that kernel, and check if vrouter-agent which is used currently, can work with that version of control.
 - If it worked well, please use in-place update, and if it won't work for some reason, or rollback operation is required, then ISSU is used

For 1, since ifmap maintains white_list for each version when importing config-api definition,
 - void IFMapGraphWalker::AddNodesToWhitelist(): https://github.com/Juniper/contrail-controller/blob/master/src/ifmap/ifmap_graph_walker.cc#L349

as far as I tried, it seems to have decent backward compatibility. (Since routing info update is similar to BGP, it also mostly should work well)

To verify this, I tried this setup with modules in different version, and it seems still working well.  
 I-1. config 2002-latest, control 2002-latest, vrouter 5.0-latest, openstack queens  
 I-2. config 2002-latest, control 5.0-latest, vrouter 5.0-latest, openstack queens  
 II-1. config 2002-latest, control 2002-latest, vrouter r5.1, kubernetes 1.12

Note: Unfortunately, this combination won't work well (cni can't get port info from vrouter-agent), I suppose this is caused by cni version change (0.2.0->0.3.1) between 5.0.x and 5.1.  
 II-2. config 2002-latest, control 2002-latest, vrouter 5.0-latest, kubernetes 1.12

So even if kernel and vRouter version don't need to be changed soon, it might be a good habit to update config / control slightly more frequently, for possible bug fix.

## L3VPN / EVPN (T2/T5, VXLAN/MPLS) integration
Before delving into this important subject, I'll firstly describe the encapsulation and control plane protocol I prefer, in two cases, which are DataCenter and NFVI.

1. DataCenter: EVPN / VXLAN
 - If you need MPLS over MPLS between DCs, you need router configuration to stitch them

2. NFVI: L3VPN / MPLS over UDP

Let me describe the reason of those choices.

### VXLAN or MPLS

To choose encapsulation, two sides, NICs and routers / switches, need to be taken care of.

For NIC side, vxlan is much more prevalent, and it is not so easy to find a hardware which can offload MPLS encap / decap, even though linux itself supports MPLS encap / decap from 4.1.
 - https://kernelnewbies.org/Linux_4.1#Multiprotocol_Label_Switching
 - If no hw offload is used, kernel vRouter will have up to 1.0 Mpps performance limit based on linux network stack, AFAIK
 - That said, let me note that vRouter currently does not support linux api to offload encap / decap of vxlan, although some configuration knob is already available: https://github.com/Juniper/contrail-specs/blob/master/smart-nic-generic-offload.md

For Router/Swtich side, it is also true that it is a bit more costly to find a hardware which can work with MPLS packets, since most of DataCenter switches currently use specific Broadcom chips, which can use vxlan, but cannot use MPLS.

So in DataCenter, to use vxlan encapsulation will be feasible.

To use VXLAN, EVPN will be the one control plane that works well.

Tungsten Fabric controller currently supports EVPN Type 2 and Type 5, and 1, 3, 4 also are used internally.
 - https://github.com/Juniper/contrail-specs/blob/master/EVPN-type-5-support-in-Contrail.md
 - https://github.com/Juniper/contrail-controller/blob/master/src/bgp/evpn/evpn_route.h#L47
 - Type 6 implementation also seems to be on the way: https://github.com/Juniper/contrail-specs/blob/master/5.1/evpn_multicast_smet.md

So it is basically OK for vRouter to join EVPN/VXLAN network, although it is not always easy to reach full interoperability.

One thing to be careful about is vRouter is capable of vxlan routing, although some switches won't have this feature.

In this setup, you might need to be a bit careful about how inter-vxlan traffic will be sent between physical switches and vRouters.
 - This document describes that behavior well: https://www.juniper.net/documentation/en_US/release-independent/solutions/information-products/pathway-pages/solutions/l3gw-vmto-evpn-vxlan-mpls.pdf

One corner case is MPLS-over-MPLS need to be used between DCs, because of advanced MPLS features like traffic engineering and link protection.

In this case, routers have to stitch EVPN/VXLAN and EVPN/MPLS, which will be achieved with those configurations.
 - https://www.juniper.net/documentation/en_US/junos/topics/concept/data-center-interconnect-evpn-vxlan-evpn-mpls-wan-overview.html

If it is used as NFVI, since Tungsten Fabric currently doesn't support service-chain with EVPN type-5, L3VPN / MPLS over UDP will be the only possible choice.
 - Note: from R1912, control / vRouter implemented service-chain based on EVPN T5 (and VXLAN), so L3VPN / MPLS over IP won't be a strict requirement: https://github.com/Juniper/contrail-specs/blob/master/R1912/bms-service-chaining.md
 - https://github.com/Juniper/contrail-specs/blob/master/EVPN-type-5-support-in-Contrail.md#control-node
 - MPLS over GRE is also ok, although it has less entropy to be used for such as LAG load-balance

Since it is prefered option to use DPDK in this case, linux stack's throughput limitation won't be an issue.

### EVPN / VXLAN interop

To illustrate the evpn / vxlan integration, let me describe L2VNI and L3VNI setup with CumulusVX (it uses FRRouting and vanilla linux's vrf / virtual-switch)
 - other sample about L3VPN / MPLS over (GRE|UDP) could be found there (TODO: config sample for EVPN / MPLS over (GRE/UDP))
 - https://marcelwiget.blog/2015/07/30/run-juniper-vmx-as-contrail-gateway-for-ipv6-overlay/

```
[1. sample setup]

Tungsten Fabric controller: 192.168.122.141/24
Tungsten Fabric vRouter: 192.168.122.142/24
 vn1 (vxlan id: 7), 10.0.1.0/24, route-target: 64512:7 is set
  10.0.1.3 is a cirros container inside vn1
  vn1 is connected to lr1 (logical-router, vxlan id: 8, route-target 64512:8 is set)
   Tungsten Fabric's project setting, 'vxlan routing: enabled' is also set (this settimg might be changed in the future)
    https://review.opencontrail.org/c/Juniper/contrail-controller/+/51833
CumulusVX: 192.168.122.151/24
 swp1: centos152 (10.0.1.152/24) is connected
  -> same l2 subnet with the container inside vRouter
 swp2: centos153 (192.168.130.153/24) is connected
  -> L3VRF will route the traffic from this to the container

[2. bgp setting]

net add bgp autonomous-system 64513
net add bgp router-id 192.168.122.151
net add bgp neighbor 192.168.122.141 remote-as 64512
net add bgp neighbor 192.168.122.141 capability extended-nexthop
net add bgp l2vpn evpn  neighbor 192.168.122.141 activate
net add bgp l2vpn evpn  advertise-all-vni
net add bgp l2vpn evpn vni 7 rd 192.168.122.151:7
net add bgp l2vpn evpn vni 7 route-target import 64512:7
net add bgp l2vpn evpn vni 7 route-target export 64512:7


cumulus@cumulus:~$ net show bgp summary
show bgp ipv4 unicast summary
=============================
BGP router identifier 192.168.122.151, local AS number 64513 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 1, using 19 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
192.168.122.141 4      64512      55      43        0    0    0 00:01:15 NoNeg

Total number of neighbors 1


show bgp ipv6 unicast summary
=============================
% No BGP neighbors found


show bgp l2vpn evpn summary
===========================
BGP router identifier 192.168.122.151, local AS number 64513 vrf-id 0
BGP table version 0
RIB entries 3, using 456 bytes of memory
Peers 1, using 19 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
192.168.122.141 4      64512      55      43        0    0    0 00:01:15            6

Total number of neighbors 1
cumulus@cumulus:~$


[3. l2vni setting]

net add bridge bridge ports vni7
net add bridge bridge vids 7
net add interface swp1 bridge pvid 7
net add vxlan vni7 vxlan id 7
net add vxlan vni7 bridge learning off
net add vxlan vni7 bridge access 7
net add vxlan vni7 bridge arp-nd-suppress on
net add vxlan vni7 vxlan local-tunnelip 192.168.122.151
net add vlan 7 ip forward off
net add vlan 7 ipv6 forward off


cumulus@cumulus:~$ net show bgp l2vpn evpn route
BGP table version is 18, local router ID is 192.168.122.151
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[ESI]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 192.168.122.142:1
*> [2]:[0]:[0]:[48]:[52:54:00:d9:db:32]
                    192.168.122.142        100             0 64512 ?
*> [2]:[0]:[0]:[48]:[52:54:00:d9:db:32]:[32]:[192.168.122.142]
                    192.168.122.142        100             0 64512 ?
*> [3]:[0]:[32]:[192.168.122.142]
                    192.168.122.142        200             0 64512 ?
Route Distinguisher: 192.168.122.142:3
*> [2]:[0]:[0]:[48]:[02:98:81:86:80:8a]
                    192.168.122.142        100             0 64512 ?
*> [2]:[0]:[0]:[48]:[02:98:81:86:80:8a]:[32]:[10.0.1.3]
                    192.168.122.142        100             0 64512 ?
*> [3]:[0]:[32]:[192.168.122.142]
                    192.168.122.142        200             0 64512 ?
Route Distinguisher: 192.168.122.142:4
*> [5]:[0]:[0]:[32]:[10.0.1.3]
                    192.168.122.142        100             0 64512 ?
 (snip)
Route Distinguisher: 192.168.122.151:7
*> [3]:[0]:[32]:[192.168.122.151]
                    192.168.122.151                    32768 i
Route Distinguisher: 192.168.122.151:8
*> [5]:[0]:[0]:[24]:[192.168.131.0]
                    192.168.122.151          0         32768 ?

Displayed 12 prefixes (12 paths)
cumulus@cumulus:~$


[root@centos152 ~]# ping 10.0.1.3
PING 10.0.1.3 (10.0.1.3) 56(84) bytes of data.
64 bytes from 10.0.1.3: icmp_seq=1 ttl=64 time=1.37 ms
64 bytes from 10.0.1.3: icmp_seq=2 ttl=64 time=0.836 ms
64 bytes from 10.0.1.3: icmp_seq=3 ttl=64 time=0.778 ms
64 bytes from 10.0.1.3: icmp_seq=4 ttl=64 time=0.753 ms
64 bytes from 10.0.1.3: icmp_seq=5 ttl=64 time=0.801 ms

--- 10.0.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 0.753/0.908/1.374/0.235 ms
[root@centos152 ~]#


cumulus@cumulus:~$ net show evpn arp-cache vni all
VNI 7 #ARP (IPv4 and IPv6, local and remote) 3

IP                        Type   State    MAC               Remote VTEP
10.0.1.152                local  active   52:54:00:20:e5:9a
fe80::28a0:caff:fe62:d16c local  active   2a:a0:ca:62:d1:6c
10.0.1.3                  remote active   02:98:81:86:80:8a 192.168.122.142
cumulus@cumulus:~$
 -> mac address of 10.0.1.3 container is learnt from Tungsten Fabric controller



[4. l3vni setting]

net add vrf vrf8 vni 8
net add bgp router-id 192.168.122.151
net add bgp vrf vrf8 autonomous-system 64513
net add bgp vrf vrf8 ipv4 unicast redistribute connected
net add bgp vrf vrf8 l2vpn evpn  advertise ipv4 unicast
net add bgp vrf vrf8 l2vpn evpn  rd 192.168.122.151:8
net add bgp vrf vrf8 l2vpn evpn  route-target import 64512:8
net add bgp vrf vrf8 l2vpn evpn  route-target export 64512:8
net add vxlan vni8 vxlan id 8
net add interface swp2 bridge pvid 8
net add vlan 8 ip address 192.168.131.254/24
net add vlan 8 vlan-id 8
net add vlan 8 vrf vrf8
net add vxlan vni8 vxlan local-tunnelip 192.168.122.151
net add vxlan vni8 bridge access 8


cumulus@cumulus:~$ net show bgp l2vpn evpn route type prefix
BGP table version is 4, local router ID is 192.168.122.151
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[ESI]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 192.168.122.142:4
*> [5]:[0]:[0]:[32]:[10.0.1.3]
                    192.168.122.142        100             0 64512 ?
Route Distinguisher: 192.168.122.151:8
*> [5]:[0]:[0]:[24]:[192.168.131.0]
                    192.168.122.151          0         32768 ?

Displayed 2 prefixes (2 paths) (of requested type)
cumulus@cumulus:~$


cumulus@cumulus:~$ net show route vrf vrf8
show ip route vrf vrf8
=======================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route


VRF vrf8:
K * 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:31:09
B>* 10.0.1.3/32 [20/100] via 192.168.122.142, vlan8 onlink, 00:31:09
C>* 192.168.131.0/24 is directly connected, vlan8, 00:29:05


[root@centos153 ~]# ping 10.0.1.3
PING 10.0.1.3 (10.0.1.3) 56(84) bytes of data.
64 bytes from 10.0.1.3: icmp_seq=1 ttl=62 time=1.27 ms
64 bytes from 10.0.1.3: icmp_seq=2 ttl=62 time=0.892 ms
64 bytes from 10.0.1.3: icmp_seq=3 ttl=62 time=0.912 ms
64 bytes from 10.0.1.3: icmp_seq=4 ttl=62 time=0.851 ms

--- 10.0.1.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 0.851/0.981/1.272/0.173 ms
[root@centos153 ~]#
[root@centos153 ~]#
[root@centos153 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.131.153/24 brd 192.168.131.255 scope global noprefixroute eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::24a9:6145:e488:5f15/64 scope link noprefixroute \       valid_lft forever preferred_lft forever
[root@centos153 ~]#
[root@centos153 ~]# ip route
default via 192.168.131.254 dev eth0 proto static metric 100
192.168.131.0/24 dev eth0 proto kernel scope link src 192.168.131.153 metric 100
[root@centos153 ~]#
```

### To configure EVPN T5 route

Before R1908, to enable EVPN T5, vxlan-routing is a project level setting, so once this knob is enabled, all the logical-router are in type: vxlan-routing, and can't be used as snat-routing logical-router.
 - https://github.com/Juniper/contrail-specs/blob/master/EVPN-type-5-support-in-Contrail.md

After R1908, this setting can be set per logical-router.
 - https://review.opencontrail.org/c/Juniper/contrail-controller/+/51794

Having said that, currently there is no way to create vxlan-routing logical-router from webui (it can be created by API).

One way to try this feature is, to modify config-api module to use vxlan-routing instead of snat-routing.
```
# docker exec -it config_api_1 bash
  # sed -i 's/snat-routing/vxlan-routing/' /usr/lib/python2.7/site-packages/vnc_cfg_api_server/resources/logical_router.py
  # exit
# docker restart config_api_1
```

After this, when some logical-router is attached to a vitual-network, EVPN T5 route is sent to other bgp peer.
 - Orchestrator needs to be openstack

```
(one VM is created in virtual-network vn1)
(kolla-toolbox)[ansible@ip-172-31-13-153 /]$ openstack server list
+--------------------------------------+------+--------+--------------+--------+---------+
| ID                                   | Name | Status | Networks     | Image  | Flavor  |
+--------------------------------------+------+--------+--------------+--------+---------+
| e3a43979-a8ae-4f05-b065-0b0841cee47b | vm1  | ACTIVE | vn1=10.0.1.3 | cirros | m1.tiny |
+--------------------------------------+------+--------+--------------+--------+---------+
(kolla-toolbox)[ansible@ip-172-31-13-153 /]$ 

(when logical-router is not connected to vn1, no type 5 route is seen)
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py ctr route show --family evpn | grep ^5
[root@ip-172-31-13-153 ~]# 


(when logical-router is connected to vn1, type 5 route for this VM is sent to other bgp peer)
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py ctr route show --family evpn | grep ^5
5-0:0-0-10.0.1.3/32, age: 0:00:07.126096, last_modified: 2020-Jan-12 13:50:27.307760
5-172.31.13.153:3-0-10.0.1.3/32, age: 0:00:07.077088, last_modified: 2020-Jan-12 13:50:27.356768
[root@ip-172-31-13-153 ~]# 
```

In addition, after R1912, EVPN T5 also can be used for service-chain route. (it can be used with vxlan)
 - https://github.com/Juniper/contrail-specs/blob/master/R1912/bms-service-chaining.md

To configure this, those procedure need to be followed.
 - tested with opencontrailnightly:1912-latest, one node install is used (openstack controller, tungsten fabric controller, vRouter is used)

1. Create two virtual-networks (vn1, vn2) and logical-routers (lr1, lr2)
2. Connect lr1 to vn1, lr2 to vn2
3. Check that virtual-network LR::lr1, LR::lr2 are automatically created
```
(kolla-toolbox)[ansible@ip-172-31-13-153 /]$ openstack network list
+--------------------------------------+-------------------------+--------------------------------------+
| ID                                   | Name                    | Subnets                              |
+--------------------------------------+-------------------------+--------------------------------------+
| 667344f9-36f1-4d56-8d9e-e5b8c856658b | LR::lr1                 | ab81f262-52d3-496f-825e-758ca5e6d60f |
| 0acf42ab-f917-4a32-a95a-5f2a555e955d | ip-fabric               |                                      |
| 5ac821b2-b823-4ea7-8be2-e1ee71547df8 | LR::lr2                 | 45b16ec8-0497-4610-843d-13d6913f4c41 |
| 0a0e30c2-d2fa-46dd-bd6f-233897f156f4 | vn1                     | c739aa67-bad3-4a69-b110-797018579b22 |
| 822b12ae-8b9c-4c32-be91-1611c245e761 | vn2                     | c67c9f25-8169-44dd-b1cd-8d9ab788a0da |
| 16715adc-93cb-4297-847a-50fcbcdef98b | __link_local__          |                                      |
| 95b08fcc-b027-407a-8b35-8470989b7d5a | dci-network             |                                      |
| 728957ed-9db3-4502-b45a-2ce3ce0ed575 | default-virtual-network |                                      |
+--------------------------------------+-------------------------+--------------------------------------+
(kolla-toolbox)[ansible@ip-172-31-13-153 /]$ 
```

4. add subnets to LR::lr1 and LR::lr2 (TF webui can be used for this)
5. create VNF with vNICs in LR::lr1 and LR::lr2
```
(kolla-toolbox)[ansible@ip-172-31-13-153 /]$ openstack server list
+--------------------------------------+------------+--------+--------------------------------------+--------+---------+
| ID                                   | Name       | Status | Networks                             | Image  | Flavor  |
+--------------------------------------+------------+--------+--------------------------------------+--------+---------+
| 4477700f-8183-4f81-b7bf-7fb16e74aba8 | vm2        | ACTIVE | vn2=10.0.2.4                         | cirros | m1.tiny |
| b631b50c-5ccf-4e48-86a8-bf390c174180 | lr1-to-lr2 | ACTIVE | LR::lr1=10.0.11.3; LR::lr2=10.0.12.3 | cirros | m1.tiny |
| e3a43979-a8ae-4f05-b065-0b0841cee47b | vm1        | ACTIVE | vn1=10.0.1.3                         | cirros | m1.tiny |
+--------------------------------------+------------+--------+--------------------------------------+--------+---------+
(kolla-toolbox)[ansible@ip-172-31-13-153 /]$ 
```
6. create service-instance, network-policy with LR::lr1 and LR::lr2, and attach network-policy to LR::lr1 and LR::lr2

When everything works fine, EVPN T5 route with protocol ServiceChain, will be added
```
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py ctr route show --family evpn | grep -e ^5 -e evpn -A 1 
default-domain:admin:__contrail_lr_internal_vn_62651c76-7851-4459-8d54-41b2b1289e21__:__contrail_lr_internal_vn_62651c76-7851-4459-8d54-41b2b1289e21__.evpn.0: 2 destinations, 2 routes (1 primary, 1 secondary, 0 infeasible)

5-0:0-0-10.0.1.3/32, age: 0:00:40.299110, last_modified: 2020-Jan-12 14:00:39.070835
    [ServiceChain (service-interface)|None] age: 0:00:40.302293, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 8, AS path: None
--
5-0:0-0-10.0.2.4/32, age: 0:04:22.046440, last_modified: 2020-Jan-12 13:56:57.323505
    [XMPP|ip-172-31-13-153.local] age: 0:04:22.049981, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 8, AS path: None
--
default-domain:admin:__contrail_lr_internal_vn_62651c76-7851-4459-8d54-41b2b1289e21__:service-20c08253-7212-40e2-8211-1548652de4b9-default-domain_admin_lr1-to-lr2.evpn.0: 2 destinations, 2 routes (1 primary, 1 secondary, 0 infeasible)

5-0:0-0-10.0.1.3/32, age: 0:00:40.299524, last_modified: 2020-Jan-12 14:00:39.070421
    [ServiceChain (service-interface)|None] age: 0:00:40.303335, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 8, AS path: None
--
5-0:0-0-10.0.2.4/32, age: 0:00:40.316583, last_modified: 2020-Jan-12 14:00:39.053362
    [XMPP|ip-172-31-13-153.local] age: 0:00:40.320727, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 8, AS path: None
--
default-domain:admin:__contrail_lr_internal_vn_7693de7f-9b96-41de-84af-c6db113132e2__:__contrail_lr_internal_vn_7693de7f-9b96-41de-84af-c6db113132e2__.evpn.0: 2 destinations, 2 routes (1 primary, 1 secondary, 0 infeasible)

5-0:0-0-10.0.1.3/32, age: 0:10:52.062185, last_modified: 2020-Jan-12 13:50:27.307760
    [XMPP|ip-172-31-13-153.local] age: 0:10:52.066796, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 6, AS path: None
--
5-0:0-0-10.0.2.4/32, age: 0:00:40.299766, last_modified: 2020-Jan-12 14:00:39.070179
    [ServiceChain (service-interface)|None] age: 0:00:40.304752, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 6, AS path: None
--
default-domain:admin:__contrail_lr_internal_vn_7693de7f-9b96-41de-84af-c6db113132e2__:service-20c08253-7212-40e2-8211-1548652de4b9-default-domain_admin_lr1-to-lr2.evpn.0: 2 destinations, 2 routes (1 primary, 1 secondary, 0 infeasible)

5-0:0-0-10.0.1.3/32, age: 0:00:40.465418, last_modified: 2020-Jan-12 14:00:38.904527
    [XMPP|ip-172-31-13-153.local] age: 0:00:40.470671, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 6, AS path: None
--
5-0:0-0-10.0.2.4/32, age: 0:00:40.299958, last_modified: 2020-Jan-12 14:00:39.069987
    [ServiceChain (service-interface)|None] age: 0:00:40.305449, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 6, AS path: None
--
default-domain:admin:vn1:vn1.evpn.0: 4 destinations, 4 routes (4 primary, 0 secondary, 0 infeasible)

--
default-domain:admin:vn2:vn2.evpn.0: 4 destinations, 4 routes (4 primary, 0 secondary, 0 infeasible)

--
bgp.evpn.0: 13 destinations, 13 routes (0 primary, 13 secondary, 0 infeasible)

--
5-172.31.13.153:3-0-10.0.1.3/32, age: 0:10:52.013177, last_modified: 2020-Jan-12 13:50:27.356768
    [XMPP|ip-172-31-13-153.local] age: 0:10:52.023700, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 6, AS path: None
--
5-172.31.13.153:5-0-10.0.2.4/32, age: 0:04:22.046385, last_modified: 2020-Jan-12 13:56:57.323560
    [XMPP|ip-172-31-13-153.local] age: 0:04:22.057108, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 8, AS path: None
--
5-172.31.13.153:6-0-10.0.2.4/32, age: 0:00:40.299816, last_modified: 2020-Jan-12 14:00:39.070129
    [ServiceChain (service-interface)|None] age: 0:00:40.310798, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 6, AS path: None
--
5-172.31.13.153:7-0-10.0.1.3/32, age: 0:00:40.299164, last_modified: 2020-Jan-12 14:00:39.070781
    [ServiceChain (service-interface)|None] age: 0:00:40.310369, localpref: 200, nh: 172.31.13.153, encap: ['vxlan'], label: 8, AS path: None
--
default-domain:default-project:ip-fabric:ip-fabric.evpn.0: 4 destinations, 4 routes (4 primary, 0 secondary, 0 infeasible)

[root@ip-172-31-13-153 ~]#
```

vRouter's vrf also will got nh to VNF
```
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py vr vrf
+--------------------------------------+---------+---------+---------+-----------+----------+--------------------------------------+
| name                                 | ucindex | mcindex | brindex | evpnindex | vxlan_id | vn                                   |
+--------------------------------------+---------+---------+---------+-----------+----------+--------------------------------------+
| default-domain:admin:__contrail_lr_i | 5       | 5       | 5       | 5         | 8        | default-domain:admin:__contrail_lr_i |
| nternal_vn_62651c76-7851-4459-8d54-4 |         |         |         |           |          | nternal_vn_62651c76-7851-4459-8d54-4 |
| 1b2b1289e21__:__contrail_lr_internal |         |         |         |           |          | 1b2b1289e21__                        |
| _vn_62651c76-7851-4459-8d54-41b2b128 |         |         |         |           |          |                                      |
| 9e21__                               |         |         |         |           |          |                                      |
| default-domain:admin:__contrail_lr_i | 7       | 7       | 7       | 7         | 0        | N/A                                  |
| nternal_vn_62651c76-7851-4459-8d54-4 |         |         |         |           |          |                                      |
| 1b2b1289e21__:service-86899929-7419  |         |         |         |           |          |                                      |
| -427a-9b3f-f8e4a3d990eb-default-     |         |         |         |           |          |                                      |
| domain_admin_lr1-to-lr2              |         |         |         |           |          |                                      |
| default-domain:admin                 | 3       | 3       | 3       | 3         | 6        | default-domain:admin                 |
| :__contrail_lr_internal_vn_7693de7f- |         |         |         |           |          | :__contrail_lr_internal_vn_7693de7f- |
| 9b96-41de-84af-c6db113132e2__        |         |         |         |           |          | 9b96-41de-84af-c6db113132e2__        |
| :__contrail_lr_internal_vn_7693de7f- |         |         |         |           |          |                                      |
| 9b96-41de-84af-c6db113132e2__        |         |         |         |           |          |                                      |
| default-domain:admin                 | 6       | 6       | 6       | 6         | 0        | N/A                                  |
| :__contrail_lr_internal_vn_7693de7f- |         |         |         |           |          |                                      |
| 9b96-41de-84af-                      |         |         |         |           |          |                                      |
| c6db113132e2__:service-86899929-7419 |         |         |         |           |          |                                      |
| -427a-9b3f-f8e4a3d990eb-default-     |         |         |         |           |          |                                      |
| domain_admin_lr1-to-lr2              |         |         |         |           |          |                                      |
| default-domain:admin:vn1:vn1         | 2       | 2       | 2       | 2         | 5        | default-domain:admin:vn1             |
| default-domain:admin:vn2:vn2         | 4       | 4       | 4       | 4         | 7        | default-domain:admin:vn2             |
| default-domain:default-project:ip-   | 0       | 0       | 0       | 0         | 0        | N/A                                  |
| fabric:__default__                   |         |         |         |           |          |                                      |
| default-domain:default-project:ip-   | 1       | 1       | 1       | 1         | 2        | default-domain:default-project:ip-   |
| fabric:ip-fabric                     |         |         |         |           |          | fabric                               |
+--------------------------------------+---------+---------+---------+-----------+----------+--------------------------------------+
[root@ip-172-31-13-153 ~]# 
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py vr route -v 3
0.255.255.252/32
    [172.31.13.153] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
    [LocalVmPort] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
10.0.1.3/32
    [EVPN-ROUTING] pref:200
     to 2:98:88:3c:38:50 via tap98883c38-50, assigned_label:-1, nh_index:34 , nh_type:interface, nh_policy:enabled, active_label:6, vxlan_id:6
10.0.2.4/32
    [172.31.13.153] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
10.0.11.0/24
    [Local] pref:100
     nh_index:1 , nh_type:discard, nh_policy:disabled, active_label:-1, vxlan_id:0
10.0.11.1/32
    [Local] pref:100
     to 0:0:0:0:0:1 via pkt0, assigned_label:-1, nh_index:13 , nh_type:interface, nh_policy:enabled, active_label:-1, vxlan_id:0
10.0.11.2/32
    [Local] pref:100
     to 0:0:0:0:0:1 via pkt0, assigned_label:-1, nh_index:13 , nh_type:interface, nh_policy:enabled, active_label:-1, vxlan_id:0
10.0.11.3/32
    [172.31.13.153] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
    [LocalVmPort] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
169.254.169.254/32
    [LinkLocal] pref:100
     via vhost0, nh_index:11 , nh_type:receive, nh_policy:enabled, active_label:0, vxlan_id:0
[root@ip-172-31-13-153 ~]# 
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py vr route -v 5
0.255.255.251/32
    [172.31.13.153] pref:200
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
    [LocalVmPort] pref:200
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
10.0.1.3/32
    [172.31.13.153] pref:200
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
10.0.2.4/32
    [EVPN-ROUTING] pref:200
     to 2:19:e0:a2:b:f3 via tap19e0a20b-f3, assigned_label:-1, nh_index:63 , nh_type:interface, nh_policy:enabled, active_label:8, vxlan_id:8
10.0.12.0/24
    [Local] pref:100
     nh_index:1 , nh_type:discard, nh_policy:disabled, active_label:-1, vxlan_id:0
10.0.12.1/32
    [Local] pref:100
     to 0:0:0:0:0:1 via pkt0, assigned_label:-1, nh_index:13 , nh_type:interface, nh_policy:enabled, active_label:-1, vxlan_id:0
10.0.12.2/32
    [Local] pref:100
     to 0:0:0:0:0:1 via pkt0, assigned_label:-1, nh_index:13 , nh_type:interface, nh_policy:enabled, active_label:-1, vxlan_id:0
10.0.12.3/32
    [172.31.13.153] pref:100
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
    [LocalVmPort] pref:100
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
169.254.169.254/32
    [LinkLocal] pref:100
     via vhost0, nh_index:11 , nh_type:receive, nh_policy:enabled, active_label:0, vxlan_id:0
[root@ip-172-31-13-153 ~]# 
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py vr route -v 6
0.255.255.252/32
    [172.31.13.153] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
10.0.2.4/32
    [172.31.13.153] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
10.0.11.3/32
    [172.31.13.153] pref:200
     to 2:34:66:61:a2:96 via tap346661a2-96, assigned_label:39, nh_index:46 , nh_type:interface, nh_policy:enabled, active_label:39, vxlan_id:0
[root@ip-172-31-13-153 ~]# 
[root@ip-172-31-13-153 ~]# 
[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py vr route -v 7
0.255.255.251/32
    [172.31.13.153] pref:200
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
10.0.1.3/32
    [172.31.13.153] pref:200
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
10.0.12.3/32
    [172.31.13.153] pref:100
     to 2:15:37:f5:fa:fb via tap1537f5fa-fb, assigned_label:44, nh_index:51 , nh_type:interface, nh_policy:enabled, active_label:44, vxlan_id:0
[root@ip-172-31-13-153 ~]# 

[root@ip-172-31-13-153 ~]# ./contrail-introspect-cli/ist.py ctr route show --family l3vpn

bgp.l3vpn.0: 9 destinations, 9 routes (0 primary, 9 secondary, 0 infeasible)

172.31.13.153:1:172.31.13.153/32, age: 0:40:32.414715, last_modified: 2020-Jan-12 13:38:26.922346
    [XMPP (interface)|ip-172-31-13-153.local] age: 0:40:32.418428, localpref: 100, nh: 172.31.13.153, encap: ['gre', 'udp', 'native'], label: 17, AS path: None

172.31.13.153:2:10.0.1.3/32, age: 0:29:55.551280, last_modified: 2020-Jan-12 13:49:03.785781
    [XMPP (interface)|ip-172-31-13-153.local] age: 0:29:55.555402, localpref: 200, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 25, AS path: None

172.31.13.153:3:0.255.255.252/32, age: 0:19:58.759556, last_modified: 2020-Jan-12 13:59:00.577505
    [XMPP (service-interface)|ip-172-31-13-153.local] age: 0:19:58.763917, localpref: 200, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 39, AS path: None

172.31.13.153:3:10.0.11.3/32, age: 0:23:22.131030, last_modified: 2020-Jan-12 13:55:37.206031
    [XMPP (interface)|ip-172-31-13-153.local] age: 0:23:22.135685, localpref: 200, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 39, AS path: None

172.31.13.153:4:10.0.2.4/32, age: 0:22:02.013695, last_modified: 2020-Jan-12 13:56:57.323366
    [XMPP (interface)|ip-172-31-13-153.local] age: 0:22:02.018717, localpref: 200, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 49, AS path: None

172.31.13.153:5:0.255.255.251/32, age: 0:19:58.547299, last_modified: 2020-Jan-12 13:59:00.789762
    [XMPP (service-interface)|ip-172-31-13-153.local] age: 0:19:58.552631, localpref: 200, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 44, AS path: None

172.31.13.153:5:10.0.12.3/32, age: 0:23:35.850393, last_modified: 2020-Jan-12 13:55:23.486668
    [XMPP (interface)|ip-172-31-13-153.local] age: 0:23:35.856031, localpref: 100, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 44, AS path: None

172.31.13.153:6:10.0.2.4/32, age: 0:08:56.528333, last_modified: 2020-Jan-12 14:10:02.808728
    [ServiceChain (service-interface)|None] age: 0:08:56.534255, localpref: 200, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 39, AS path: None

172.31.13.153:7:10.0.1.3/32, age: 0:08:56.527653, last_modified: 2020-Jan-12 14:10:02.809408
    [ServiceChain (service-interface)|None] age: 0:08:56.533918, localpref: 200, nh: 172.31.13.153, encap: ['gre', 'udp'], label: 44, AS path: None
[root@ip-172-31-13-153 ~]# 
```

### vlan-based and vlan-aware for EVPN T2

In EVPN T2, there are two flavors vlan-based, and vlan-aware, and they are mutually incompatible.

Tungsten Fabric controller, by default, uses vlan-aware flavor, so their evpn t2 route can't be imported by several datacenter switches, which only supports vlan-based flavor.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1781102

Having said that, this patch (and container based on R1912) makes ethernet tag id zero, and it is reported that some switches begin importing T2 route, if this is applied
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricKnowledgeBase.md#vlan-base-interop
 - https://hub.docker.com/r/tnaganawa/contrail-controller-control-control

## Service-Chain (L2, L3, NAT), BGPaaS

### service-chain

Although it has a lot of usecases, NFVI will be one of Tungsten Fabric's most prominent usecase, because of a lot of unique features which makes NFVI implementation software based.

Most well-known feature of this line is service-chain, which is a feature to manage traffic without changing VNF's ip's, which makes realtime insertion and remove of VNF possible.

Since vRouter can have VRFs inside, it can have VRFs at the every interface of VNF, and can make the traffic handled by the fabricated next-hops, to send such as next VNF.

Tungsten Fabric's service-chain is implemented that way, so you will see several VRFs are created once service-chain is created, and next-hop will be inserted to send traffic to the next VNF of the chain.
 - VRFs (routing-instance in control's term) are named as domain-name:project-name:virtual-network-name:routing-instance-name. In most cases, virtual-network-name and routing-instance-name is the same, but the service-chain is one exception of this rule

To set up sample service-chain, the procedure in this movie can be followed
 - https://www.youtube.com/watch?v=h5qOqsPtQ7M

After that, you can see left virtual-network has right virtual-network's prefixes with updated next-hop, which is oriented to left interface of VNF, and vice versa for right virtual-network.

Note: When service-chain v2 is used, only 'left' and 'right' interfaces are used for service-chain calculation, and 'management' and 'other' interfaces are omitted, AFAIK


### l2, l3, nat

There are a lot of VNFs with different set of traffic type, so NFVI's SDN also needs to support several type of traffic.

For this purpose, Tungsten Fabric service-chain supports three traffic type, namely l2, l3, nat.

l2 service-chain (also known as transparent service-chain) can be used with transparent VNF, which has similar feature with bridge, with sending packets based on the arp response. 

Although vRouter always use the same mac address (00:01:00:5e:00:00),
 - https://github.com/Juniper/contrail-controller/wiki/Contrail-VRouter-ARP-Processing#vrouter-mac-address

this case is an exception of this rule, and the vRouter at the left of VNF sends traffic with dest mac: 2:0:0:0:0:2, and the vRouter at the right of VNF sends traffic with dest mac 1:0:0:0:0:1.
So bridge-type VNF will send traffic to the opposite side of its interfaces.

Let me note that even if l2 vnf is used, the left virtual-network and right virtual-network need to have different subnet.
This might be a bit counter intuitive, but since vRouter can do l3 routing, vRouter - L2VNF - vRouter is possible, just like router - L2VNF - router is acceptable.


l3 service-chain (also known as in-network service-chain), on the other hand, will send traffic to the VNF without changing mac address, since in this case, VNF will route packets based on its destination ip (similar behavior with router).
Except for the mac address, the behavior is mostly the same with l2 case.


Nat service-chain is similar to l3 service-chain, since it expects VNF to route packets based on destination ip.
One big difference is it replicates right virtual-network's prefixes to left virtual-network, but it won't replicate left virtual-network's prefixes to right virtual-network!
 - so left / right interfaces need to be chosen carefully, since it's asymmetric in this case

Typical usecase of this flavor of service-chain is VNF's left interface has private ip, and the right interface has global ip, in a case such as SNAT for internet access is performed.
Since private ip can't be exported to the internet, in this case, left virtual-network's prefix can't be replicated to right virtual-network.

### ECMP, multi VNF

Service-chain feature also supports ECMP setup for scale out deployment.
 - Configuration is mostly the same, but several port-tuple need to be assigned to one service-instance.

After that, you will notice that traffic will be load-balanced based on 5-tuple of the packets.

Multi VNF also can be set, if several service-instances are assigned to one network-policy.

When l3 service-chain is used, although it might be counter intuitive, two VNFs need to be assigned to the same virtual-network.
 - Since all the packets from VNFs will be in the separate VRFs for service-chain, they can have the same subnets.

The simultaneous use of l2 and l3 is also supported, although in that case, l2 vnf needs to be assigned to the different virtual-networks, with the one network-policy is attached
 - setup example is described in this blog post: https://tungsten.io/building-and-testing-layer2-service-images-for-opencontrail/


### BGPaaS

BGPaaS is also a bit unique feature of Tungsten Fabric, which is used to insert VRFs' routes in VNFs.
 - in a sense, it is a bit similar to AWS's VPN gateway, since it automatically got the routes from VPC's route table

From operational perspective, VNFs in vRouter will have IPV4 bgp peer with vRouter's gateway ip and service ip.

One notable usecase will be to set up ipsec VNFs, which might have a connection to public cloud with VPN gateway.
In this case, VPC's route table will be copied to VNFs, and it will be replicated to vRouter's VRF through BGPaaS, so all the prefixes are distributed correctly when subnets are newly added modified in public cloud's VPC.

### subinterface

This is also a feature used in NFVI, so let me mention this here also.

VNF sends tagged packets for various reason. In this case, vRouter can use different VRFs if vlan tags are different.
 - similar to subinterfaces in 'set routing-instances routing-interface-name interface xxx' in junos's term

Operation is described there
https://www.youtube.com/watch?v=ANhBQe_DS2E


## Multicluster

Since it uses MPLS-VPN internally, virtual-networks in Tunsten Fabric can be extended to other Tungsten Fabric clusters.
 - it might be a bit surprising, since neutron ML2 plugin or some other CNI won't support this setup, AFAIK

That said, since they have different DBs, shared resources need to be marked between them.

I'll describe the usage of several bgp parameters for this purpose.

#### Routing

Since Tungsten Fabric uses L3VPN for inter-VRF routing, if route-target is correctly set between VRFs, it can route packets.
 - Since network-policy / logical-router can not be used between several clusters, route-targets need to be directly configured on each virtual-network.

Note: if l3-only forwarding is specified, even in intra-VRF forwarding, L3VPN is used, so bridging won't be used in that setup.

### Bridging

### security-group

Tungsten Fabric also have some extended community to convey security-group id.
 - https://github.com/Juniper/contrail-controller/wiki/BGP-Extended-Communities

Since this id also can be manually configured, you can set the same id to each cluster's security-group, and to allow traffic from that prefix.

Note: As far as I tried, tags' id can't be manually configured from Tungsten Fabric webui in R5.1 branch, so fw-policy can't be used between clusters. This behavior might be changed in future.

### DNS

DNS is an important subject when dealing with several clusters.

Since Tungsten Fabric have vDNS implementation similar to openstack's default setup, you can resolve vmname in a cluster, and make those names available externally.
 - https://github.com/Juniper/contrail-controller/wiki/DNS-and-IPAM
 - Controller nodes has a process contrail-named, to respond to the external DNS query
 - To enabled this, from Tungsten Fabric webui, Configure > DNS > DNS Server > (create) > External Access need to be checked

So at least when openstack (or vCenter) is used as an orchestrator, and if different clusters have different domain names, it can directly resove the names of other clusters.
 - Upstream DNS forwarder need to be able to resolve all the name

When kubernetes is used, Tungsten Fabric use coredns as the source of name resolusion, rather than on its own vDNS. Those IPs and domain names can be changed in kubeadm setting.
```
cluster0:
kubeadm init --pod-network-cidr=10.32.0.0/24 --service-cidr=10.96.0.0/24
cluster1:
kubeadm init --pod-network-cidr=10.32.1.0/24 --service-cidr=10.96.1.0/24 --service-dns-domain=cluster1.local

cluster1:
# cat /etc/sysconfig/kubelet 
-KUBELET_EXTRA_ARGS=
+KUBELET_EXTRA_ARGS="--cluster-dns=10.96.1.10"
# systemctl restart kubelet
```

Note: When it is configured, Tungsten Fabric setting also need to be changed (set in configmap env)
```
cluster0:
  KUBERNETES_POD_SUBNETS: 10.32.0.0/24
  KUBERNETES_IP_FABRIC_SUBNETS: 10.64.0.0/24
  KUBERNETES_SERVICE_SUBNETS: 10.96.0.0/24

cluster1:
  KUBERNETES_POD_SUBNETS: 10.32.1.0/24
  KUBERNETES_IP_FABRIC_SUBNETS: 10.64.1.0/24
  KUBERNETES_SERVICE_SUBNETS: 10.96.1.0/24
```

After setting coredns, it can resolve the name of other clusters (coredns IPs need to be leaked to each other's VRF, since those IPs need to be reachable)
```
kubectl edit -n kube-system configmap coredns

cluster0:
### add these lines to resolve cluster1 names
    cluster1.local:53 {
        errors
        cache 30
        forward . 10.96.1.10
    }
    
cluster1:
### add these lines to resolve cluster0 names
    cluster.local:53 {
        errors
        cache 30
        forward . 10.96.0.10
    }
    
```

So even if you have several separate Tungsten Fabric clusters, it is not too difficult to stitch virtual-networks between them.

To have larger number of nodes than orchestrator currently supports, could be one reason to do so, even though orchestrators like kubernetes, openstack, vCenter support fairly large number of hypervisors.

### Inter AS option B/C

## Multi-DC

If traffic is around multi DCs, you need to be a bit careful when planning Tungsten Fabric installation.

There are two options: 1. single cluster, 2. multi clusters.

Single cluster option is simpler and easier to manage, although RTT between DCs could be an issue, since several traffic such as XMPP, rabbitmq, cassandra will go through controllers (Currently, locality support is not available around them)

Multi cluster approach will give a bit more operational complexity, since both clusters have different DBs, you need to manually set some parameters, such as route-targets or security-group ids.

Additionally, vMotion between them also will be much more difficult.
 - Even if cross vCenter vMotion is used, since the new vCenter and new Tungsten Fabric cluster will create a new port, it would have different fixed ip with the original one.
 - Nova won't support cross openstck live migration currently, so if openstack is used, it is not possible to do live migration between them

Since vCenter requires 150ms RTT between DCs (I couldn't find similar value for KVM), single cluster < 150 msec RTT < multi clusters might be one rule of thumb, although it has to be planned carefully for each specific case.
 - https://kb.vmware.com/s/article/2106949

When single cluster installation is planned, and the number of DCs are two, one thing additionally need to be cared.

Since zookeeper / cassandra in Tungsten Fabric currently use Quorum consistency level, when primary site is down, second site can't keep working. (Both of Read and Write access will be unavaiable)
 - https://github.com/Juniper/contrail-controller/blob/master/src/config/common/vnc_cassandra.py#L659 (used by config-api, schema-transformer, svc-monitor, device-manager)
 - https://github.com/Juniper/contrail-common/blob/master/config-client-mgr/config_cassandra_client.cc#L458 (used by control, dns)

One possible option to workaround this is to change consistency level to ONE / TWO / THREE or LOCAL_ONE / LOCAL_QUORUM, although it needs rebuild of source code.

Since zookeeper has no such knob, the only way I'm aware of is to update weight, after the primary site is down.
 - https://stackoverflow.com/questions/32189618/hierarchical-quorums-in-zookeeper
 - Most of the components continue working even if zookeeper is temporary unavaialble, although components it uses that for HA stop working (schema-transformer, svc-monitor, kube-manager, vcenter-plugin, ...)

When number of DCs are over two, this won't be an issue.


## Multi orchestrator

Sharing controll plane between several orchestrators do a lot of good thing, including routing/bridging, DNS, security, ..

Let me describe the usage and configuration about each scenario.

### k8s+openstack

kubernetes + openstack combination is already covered and works well.
 - https://github.com/Juniper/contrail-ansible-deployer/wiki/Deployment-Example:-Contrail-and-Kubernetes-and-Openstack

One additional comment is Tungsten Fabric supports both of nested installation and non-nested installation, so you can choose either option.
 - https://github.com/Juniper/contrail-kubernetes-docs
 
### k8s+k8s
To add multiple kubernetes cluster to one Tungsten Fabric could be one installation option.

Since kube-manager supports one parameter cluster_name, which modifies the tenant name which will be created (default is 'k8s'), it is likely that can be OK, although when I tried that last time it doesn't work well, since some objects was deleted by other kube-manager as the stale objects.
 - https://github.com/Juniper/contrail-container-builder/blob/master/containers/kubernetes/kube-manager/entrypoint.sh#L28

This behavior might be changed in future release.

Note:  
From R2002 and later, this patch fixed the issue and custom patch is not needed anymore.
 - https://review.opencontrail.org/c/Juniper/contrail-controller/+/55758
 
Note:
Applying these patch, it seems possible to add multiple kube-master to one Tungsten Fabric cluster.

```
diff --git a/src/container/kube-manager/kube_manager/kube_manager.py b/src/container/kube-manager/kube_manager/kube_manager.py
index 0f6f7a0..adb20a6 100644
--- a/src/container/kube-manager/kube_manager/kube_manager.py
+++ b/src/container/kube-manager/kube_manager/kube_manager.py
@@ -219,10 +219,10 @@ def main(args_str=None, kube_api_skip=False, event_queue=None,
 
     if args.cluster_id:
         client_pfx = args.cluster_id + '-'
-        zk_path_pfx = args.cluster_id + '/'
+        zk_path_pfx = args.cluster_id + '/' + args.cluster_name
     else:
         client_pfx = ''
-        zk_path_pfx = ''
+        zk_path_pfx = '' + args.cluster_name
 
     # randomize collector list
     args.random_collectors = args.collectors
diff --git a/src/container/kube-manager/kube_manager/vnc/vnc_namespace.py b/src/container/kube-manager/kube_manager/vnc/vnc_namespace.py
index 00cce81..f968cae 100644
--- a/src/container/kube-manager/kube_manager/vnc/vnc_namespace.py
+++ b/src/container/kube-manager/kube_manager/vnc/vnc_namespace.py
@@ -594,7 +594,8 @@ class VncNamespace(VncCommon):
                 self._queue.put(event)
 
     def namespace_timer(self):
-        self._sync_namespace_project()
+        # self._sync_namespace_project() ## temporary disabled
+        pass
 
     def _get_namespace_firewall_ingress_rule_name(self, ns_name):
         return "-".join([vnc_kube_config.cluster_name(),

```

Since both kube-masters create pod-networks to the same Tungsten Fabric controller, route-leak between them would be possible :)
 - Since cluster_name will be one of the tags in Tungsten Fabric's fw-policy, it also would be possible to use same tags between multiple kubernetes clusters
```
172.31.9.29 Tungsten Fabric controller
172.31.22.24 kube-master1 (KUBERNETES_CLUSTER_NAME=k8s1 is set)
172.31.12.82 kube-node1 (it belongs to kube-master1)
172.31.41.5 kube-master2(KUBERNETES_CLUSTER_NAME=k8s2 is set)
172.31.4.1 kube-node2 (it belongs to kube-master2)


[root@ip-172-31-22-24 ~]# kubectl get node
NAME                                              STATUS     ROLES    AGE   VERSION
ip-172-31-12-82.ap-northeast-1.compute.internal   Ready      <none>   57m   v1.12.3
ip-172-31-22-24.ap-northeast-1.compute.internal   NotReady   master   58m   v1.12.3
[root@ip-172-31-22-24 ~]# 

[root@ip-172-31-41-5 ~]# kubectl get node
NAME                                             STATUS     ROLES    AGE   VERSION
ip-172-31-4-1.ap-northeast-1.compute.internal    Ready      <none>   40m   v1.12.3
ip-172-31-41-5.ap-northeast-1.compute.internal   NotReady   master   40m   v1.12.3
[root@ip-172-31-41-5 ~]# 

[root@ip-172-31-22-24 ~]# kubectl get pod -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE                                              NOMINATED NODE
cirros-deployment-75c98888b9-7pf82   1/1     Running   0          28m     10.47.255.249   ip-172-31-12-82.ap-northeast-1.compute.internal   <none>
cirros-deployment-75c98888b9-sgrc6   1/1     Running   0          28m     10.47.255.250   ip-172-31-12-82.ap-northeast-1.compute.internal   <none>
cirros-vn1                           1/1     Running   0          7m56s   10.0.1.3        ip-172-31-12-82.ap-northeast-1.compute.internal   <none>
[root@ip-172-31-22-24 ~]# 


[root@ip-172-31-41-5 ~]# kubectl get pod -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE                                            NOMINATED NODE
cirros-deployment-75c98888b9-5lqzc   1/1     Running   0          27m     10.47.255.250   ip-172-31-4-1.ap-northeast-1.compute.internal   <none>
cirros-deployment-75c98888b9-dg8bf   1/1     Running   0          27m     10.47.255.249   ip-172-31-4-1.ap-northeast-1.compute.internal   <none>
cirros-vn2                           1/1     Running   0          5m36s   10.0.2.3        ip-172-31-4-1.ap-northeast-1.compute.internal   <none>
[root@ip-172-31-41-5 ~]# 


/ # ping 10.0.2.3
PING 10.0.2.3 (10.0.2.3): 56 data bytes
64 bytes from 10.0.2.3: seq=83 ttl=63 time=1.333 ms
64 bytes from 10.0.2.3: seq=84 ttl=63 time=0.327 ms
64 bytes from 10.0.2.3: seq=85 ttl=63 time=0.319 ms
64 bytes from 10.0.2.3: seq=86 ttl=63 time=0.325 ms
^C
--- 10.0.2.3 ping statistics ---
87 packets transmitted, 4 packets received, 95% packet loss
round-trip min/avg/max = 0.319/0.576/1.333 ms
/ # 
/ # ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue \    link/ether 02:b9:11:c9:4c:b1 brd ff:ff:ff:ff:ff:ff
18: eth0    inet 10.0.1.3/24 scope global eth0\       valid_lft forever preferred_lft forever
/ #
 -> ping between pods, which belong to different kubernetes clusters, worked well


[root@ip-172-31-9-29 ~]# ./contrail-introspect-cli/ist.py ctr route show -t default-domain:k8s1-default:vn1:vn1.inet.0 

default-domain:k8s1-default:vn1:vn1.inet.0: 2 destinations, 2 routes (1 primary, 1 secondary, 0 infeasible)

10.0.1.3/32, age: 0:06:50.001343, last_modified: 2019-Jul-28 18:23:08.243656
    [XMPP (interface)|ip-172-31-12-82.local] age: 0:06:50.005553, localpref: 200, nh: 172.31.12.82, encap: ['gre', 'udp'], label: 50, AS path: None

10.0.2.3/32, age: 0:02:25.188713, last_modified: 2019-Jul-28 18:27:33.056286
    [XMPP (interface)|ip-172-31-4-1.local] age: 0:02:25.193517, localpref: 200, nh: 172.31.4.1, encap: ['gre', 'udp'], label: 50, AS path: None
[root@ip-172-31-9-29 ~]# 
[root@ip-172-31-9-29 ~]# ./contrail-introspect-cli/ist.py ctr route show -t default-domain:k8s2-default:vn2:vn2.inet.0 

default-domain:k8s2-default:vn2:vn2.inet.0: 2 destinations, 2 routes (1 primary, 1 secondary, 0 infeasible)

10.0.1.3/32, age: 0:02:36.482764, last_modified: 2019-Jul-28 18:27:33.055702
    [XMPP (interface)|ip-172-31-12-82.local] age: 0:02:36.489419, localpref: 200, nh: 172.31.12.82, encap: ['gre', 'udp'], label: 50, AS path: None

10.0.2.3/32, age: 0:04:37.126317, last_modified: 2019-Jul-28 18:25:32.412149
    [XMPP (interface)|ip-172-31-4-1.local] age: 0:04:37.133912, localpref: 200, nh: 172.31.4.1, encap: ['gre', 'udp'], label: 50, AS path: None
[root@ip-172-31-9-29 ~]#
 -> each virtual-network in each kube-master has a route to other kube-master's pod, based on network-policy below



(venv) [root@ip-172-31-9-29 ~]# contrail-api-cli --host 172.31.9.29 ls -l virtual-network
virtual-network/f9d06d27-8fc1-413d-a6d6-c51c42191ac0  default-domain:k8s2-default:vn2
virtual-network/384fb3ef-247b-42e6-a628-7111fe343f90  default-domain:k8s2-default:k8s2-default-service-network
virtual-network/c3098210-983b-46bc-b750-d06acfc66414  default-domain:k8s1-default:k8s1-default-pod-network
virtual-network/1ff6fdbd-ac2e-4601-b08c-5f7255466312  default-domain:default-project:ip-fabric
virtual-network/d8d95738-0a00-457f-b21e-60304859d1f9  default-domain:k8s2-default:k8s2-default-pod-network
virtual-network/0c075b76-4219-4f79-a4f5-1b4e6729f16e  default-domain:k8s1-default:k8s1-default-service-network
virtual-network/985b3b5f-84b7-4810-a54d-abd09a37f525  default-domain:k8s1-default:vn1
virtual-network/23782ea7-4000-491f-b20d-01c6ab9e2ba8  default-domain:default-project:default-virtual-network
virtual-network/90cce352-ef9b-4358-81b3-ef87a9cb63e8  default-domain:default-project:__link_local__
virtual-network/0292810c-c511-4147-89c0-9fdd571ccce8  default-domain:default-project:dci-network
(venv) [root@ip-172-31-9-29 ~]# 

(venv) [root@ip-172-31-9-29 ~]# contrail-api-cli --host 172.31.9.29 ls -l network-policy
network-policy/134d38b2-79e2-4a3e-a2f7-a3d61ceaf5e2  default-domain:k8s1-default:vn1-to-vn2  <-- route-leak between to kubernetes cluster
network-policy/8e5c5c4a-14eb-4fc4-9b46-81a5b923bbe0  default-domain:k8s1-default:k8s1-default-ip-fabric-np
network-policy/544d5076-3dff-45a1-bb47-8aec5e1e5a37  default-domain:k8s1-default:k8s1-default-pod-service-np
network-policy/33884d88-6492-4e0f-934c-080a794ce132  default-domain:k8s2-default:k8s2-default-ip-fabric-np
network-policy/232beb43-2008-4df3-969f-a4eee653ff46  default-domain:k8s2-default:k8s2-default-pod-service-np
network-policy/a6ee02bd-ad0d-4393-be60-66da8032237a  default-domain:k8s2-default:k8s2-default-service-np
network-policy/a9cedd67-127a-40fd-9f44-78890dc3cfe4  default-domain:k8s1-default:k8s1-default-service-np
(venv) [root@ip-172-31-9-29 ~]#
```

### openstack+openstack

I haven't yet tried to add two openstack clusters to one Tungsten Fabric controller, but it might be possible if they don't use same tenant name.

### k8s+vCenter
Kubernetes and vCenter combination could be used simultaneously. Usecase is similar to kubernetes+openstack.

### openstack+vCenter

Openstack and vCenter combination is a bit curious, since openstack dashboard might be used as the management UI for vCenter network.

As far as I tried, vcenter-plugin checks all the virtual-networks under all avaiable tenants, rather than virtual-networks under 'vCenter' tenant, so if virtual-network or other neutron components are created, that also can be available at vRouterVM on ESXi. With this setup, vCenter users can implement network function by themselves, just like their using EC2 / VPC.
 - They can also use permission feature of vCenter, to implement pseudo multi-tenancy of VMI and NF.
 - https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.security.doc/GUID-4B47F690-72E7-4861-A299-9195B9C52E71.html


### vCenter+vCenter

Multi vCenter is an important subject, since vCenter has well defined configuration maximums and multi vCenter installation is a common way to work around them.

Simplest setting in this case is to configure multi Tungsten Fabric cluters per vCenter, but in that case it will be diffcult to do vMotion between two clusters, since Tungsten Fabric create a new port after vMotion finished, and might assign different fixed ip.

So I think assigning several vCenters to one Tungsten Fabric cluster would have legitimate usecase.

As far as I tried, in current implementation, since vcenter-plugin uses only 'vCenter' tenant for some objects, it is not possible to use two vcenter-plugin simultaneously, without some code modification.

If tenants can be modified per vcenter-plugin and vcenter-manager, it might be possible to assign each vCenter a separate tenant, and use them simultaneously, just like use kubernetes and openstack simultaneously.

If this were available, it also will be possible to use service-insertion and physical switch extenstion with multi-vCenter environment.
 - Even SRM integration also might be on that way, since place holder VM will assign a new port, which can be editted to assign correct fixed ip

### k8s+openstack+vCenter

I don't know if this configuration will be ever used, since kubernetes / openstack / vCenter have some feature overlap, although it would work well if set up.

## DPDK

vRouter has a feature to use DPDK to interact with physical NIC.

It will be frequently used for NFV type deployment, since it is still not easy to have forwarding performance comparable to typical VNF (which itself might use DPDK or similar technology), based pure linux kernel networking stack.
 - https://blog.cloudflare.com/how-to-receive-a-million-packets/

To enable this feature with ansible-deployer, those parameters need to be set.
```
bms1:
  roles:
    vrouter:
      AGENT_MODE: dpdk
      CPU_CORE_MASK: “0xf” ## coremask for forwarding core (Note: this means 0-3, but please don't include first core in numa to reach optimal performance :( )
      DPDK_UIO_DRIVER: uio_pci_generic ## uio driver name
      HUGE_PAGES: 16000 ## number of 2MB hugepages, it can be smaller
```

When AGENT_MODE: dpdk is set, ansible-deployer will install some containers such as vrouter-dpdk, which is a process to run PMD against physical NIC, so in that case, forwarding from vRouter to physical NIC will be based on DPDK.

Note:
 1. Since vRouter is linked to limited number of PMDs, to use some specific NIC, vRouter rebuild might be needed
  - https://github.com/Juniper/contrail-vrouter/blob/master/SConscript#L321
 2. For some NIC such as XL710, uio_pci_generic can't be used. In that case, vfio-pci need to be used instead
  - https://doc.dpdk.org/guides-18.05/rel_notes/known_issues.html#uio-pci-generic-module-bind-failed-in-x710-xl710-xxv710

Since in that case, vRouter's forwarding plane is not in kernel space, tap device can't be used to get the packets from VMs.
For this purpose, QEMU has a feature 'vhostuser', to send packets to dpdk process in user space.
When vRouter is configured with AGENT_MODE: dpdk, nova-vif-driver automatically create vhostuser vif, rather than tap vif, which is used for kernel vRouter
 - From VM side, it still looks like virtio, so usual virtio driver can be used to communicate with DPDK vRouter.

One caveat is that when QEMU will be connected to vhostuser interface, qemu also need to have hugepage for that.
When openstack is used, this knob will assign hugepage to each VM.
```
openstack flavor set flavorname --property hw:mem_page_size=large 
 - hw:mem_page_size=2MB, hw:mem_page_size=1GB also can be used
```

To reach optimal performance, there are a lot of tuning parameters, both in kernel and dpdk process itself.
From kernel side, for me, these two articles are most helpful.
 - https://www.redhat.com/en/blog/tuning-zero-packet-loss-red-hat-openstack-platform-part-1
 - https://www.redhat.com/en/blog/going-full-deterministic-using-real-time-openstack
 - cat /proc/sched_debug also can be used to see if core isolation is working well

From vRouter side, this point might need to be taken care of.
 1. vRouter will use core load-balance based on 5-tuple, so for optimal performance, number of flows might need to be increased
  - https://www.openvswitch.org/support/ovscon2018/6/0940-yang.pptx

## Service Mesh
istio is working well, multicluster could be interesting subject
 - https://www.youtube.com/watch?v=VSNc9qd2poA
 - https://istio.io/docs/setup/kubernetes/install/multicluster/vpn/
