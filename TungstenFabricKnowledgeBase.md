

Table of Contents
=================

   * [Tungsten Fabric Knowledge Base](#tungsten-fabric-knowledge-base)
      * [vRouter internal](#vrouter-internal)
      * [control internal](#control-internal)
      * [config internal](#config-internal)
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

## vRouter scale test procedure

Since GCP allowed me to start up to 5k nodes :), this procedure is described primarily for this platform.
 - Having said that, the same procedure also can be used with AWS

First target was 3k vRouter nodes, but as far as I tried, it is not the maximum, and more might be done with more control nodes or CPU/MEM are added.

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

For the later use, virtual-network vn1 (10.0.1.0/24, l2/l3) is also created.
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

It takes about 20-30 minutes.

4. Install kubernetes on vRouter nodes, and wait for vRouter nodes are installed by that.
```
(/tmp/aaa.pem is the secret key for GCP)
sudo yum -y install epel-release
sudo yum -y install parallel
ulimit -n 8192
cat aaa.txt | parallel -j3500 scp -i /tmp/aaa.pem -o StrictHostKeyChecking=no install-k8s-packages.sh centos@{}:/tmp
cat aaa.txt | parallel -j3500 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} chmod 755 /tmp/install-k8s-packages.sh
cat aaa.txt | parallel -j3500 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} sudo /tmp/install-k8s-packages.sh
cat aaa.txt | parallel -j3500 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} sudo kubeadm join 172.16.1.x:6443 --token we70in.mvy0yu0hnxb6kxip --discovery-token-ca-cert-hash sha256:13cf52534ab14ee1f4dc561de746e95bc7684f2a0355cb82eebdbd5b1e9f3634
```

It takes about 20-30 minutes.


5. after that, first-containers.yaml can be created with replica: 3000, and ping between containers can be checked.
 To see BUM behavior, vn1 also can be used with containers with 3k replicas.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/8-leaves-contrail-config.txt#L99

vRouter installation takes about 20 minutes.
Container creation might take longer, such as up to 1hr, since IP assignment for each container takes some time .. :(

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

When installed on public cloud, vRouter needs to have floating-ip from underlay IP, since no hardware which serve MPLS over IP or VXLAN is avalaible ..

Fortunately, since 4.1, tungsten fabric supports gatewayless feature, so it won't have much difficulty to serve floating-ip from that virtual-network (idea is to attach another IP with ENI, and make it the source of floating-ip, to make the service on vRouter accessible from the outside world)
 - https://github.com/Juniper/contrail-specs/blob/master/gateway-less-forwarding.md

From vRouter to the outside world, Distribute SNAT feature would do the trick.
 - https://github.com/Juniper/contrail-specs/blob/master/distributed-snat.md
 - SNAT with floating-ip also would work well

Additionally, it would be also possible to define two separate load balancers on vRouters to reach the same application, to make it accessble from two different avaibility zones, which potentially ensure higher availablity.
 - to be investigated

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

By default, vRouter CNI uses thick-plugin mode, and it can assign multiple vRouter vnic to a container by itself.
https://github.com/Juniper/contrail-specs/blob/master/5.1/K8s_pods_multiple_interfaces.md

Unfortunately, it is not compabtible with multus (but I haven't yet fully tested), so when it is used with other CNI (such as SR-IOV, bridge, ..), meta-plugin mode (by default, false is used) need to be specified.
 - I guess if only one vNIC is used, meta-plugin parameter won't change something, but I haven't yet tested.

```
(contrail-container-builder)

diff --git a/containers/kubernetes/cni-init/entrypoint.sh b/containers/kubernetes/cni-init/entrypoint.sh
index 01a4698..18d36c8 100755
--- a/containers/kubernetes/cni-init/entrypoint.sh
+++ b/containers/kubernetes/cni-init/entrypoint.sh
@@ -41,7 +41,8 @@ cat << EOM > /host/etc_cni/net.d/10-contrail.conf
         "poll-timeout"  : 5,
         "poll-retries"  : 15,
         "log-file"      : "$LOG_DIR/cni/opencontrail.log",
-        "log-level"     : "4"
+        "log-level"     : "4",
+        "meta-plugin"     : true
     },
 
     "name": "contrail-k8s-cni",
```

Note: when i tried with 2002-latest, it returns this error, even when this knob is set to 'false' :(

```
I : 13192 : 2020/02/08 13:52:13 vrouter.go:223: Get from vrouter passed. Result &[{VmUuid: Nw: Ip: Plen:0 Gw: Dns: Mac:02:26:3a:9e:14:4a VlanId:65535 VnId:e38b4a74-a95e-4d1e-8166-4569090472b2 VnName:default-domain:k8s-default:k8s-default-pod-network VmiUuid:263a9e14-4a7a-11ea-9efd-063cf1da2350 Args:[{cluster:k8s} {index:0/2} {kind:Pod} {name:vlan-172.16.1.201} {namespace:default} {network:default} {owner:k8s} {project:k8s-default}] Annotations:{Cluster:k8s Kind:Pod Name:vlan-172.16.1.201 Namespace:default Network:default Owner:k8s Project:k8s-default Index:0/2 Interface:}}]
I : 13192 : 2020/02/08 13:52:13 vrouter.go:511: Iteration 4 : Get VRouter Incomplete - Interfaces Expected: 2, Actual: 1
```

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

