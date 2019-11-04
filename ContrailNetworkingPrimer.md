
Table of Contents
=================

   * [Contrail Command](#contrail-command)
   * [Contrail Fabric Automation](#contrail-fabric-automation)
      * [vQFX setting](#vqfx-setting)
      * [LACP against linux bridge](#lacp-against-linux-bridge)
      * [vQFX limitation](#vqfx-limitation)
      * [Integration with fabric automation and vRouters](#integration-with-fabric-automation-and-vrouters)
      * [PNF integration](#pnf-integration)
      * [Ironic integration](#ironic-integration)
      * [Appformix integration](#appformix-integration)
   * [Contrail Healthbot](#contrail-healthbot)
   * [Contrail multicloud](#contrail-multicloud)
      * [container deployment](#container-deployment)
      * [baremetal instance deployment](#baremetal-instance-deployment)
   * [Contrail SD-WAN](#contrail-sd-wan)


This document is to describe the difference between Contrail Networking and Tungsten Fabric.

At its core, contrail networking uses the same module with Tungsten Fabric (control, vRouter, config, ...), so they are not much different technically.

The main difference is contrail networking also has features to extend the virtual network to several places, such as physical device, sd-branch, and cloud VPC, to make virtual-network concept more versatile.

Let me describe each feature of contrail networking.

## Contrail Command

To manage all of them, contrail package has a separate webui named contrail-command.
![ClusterView](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/CommandClusterView.png)
![MenuView](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/CommandMenuView.png)

It covers several topics including all the features of Tungsten Fabric webui, and fabric automation feature and multicloud extenstion feature.

To install that, you can firstly set up Tungsten Fabric controller, in some way described in this doc. (tungstenfabirc, r5.1 might work well, although latest module of monthly relase (R1907-latest, R1908-latest, ..) might be recommended)
https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#2-tungstenfabric-up-and-running

 - Note: to make fabric automation without openstack, you might need this knob: https://review.opencontrail.org/c/Juniper/contrail-controller/+/51846
 ```
   DEVICE_MANAGER__KEYSTONE__admin_password: contrail123
   API__KEYSTONE__admin_password: contrail123
 ```

After that, you can import those controllers based on these two files (all of kubernetes, openstack, vCenter will work well with contrail-command)
 - Note: for kubernetes or vCenter, you need this knob: auth_type: basic-auth (for openstack this knob is not needed)
```
export docker_registry=hub.juniper.net/contrail
export container_tag=1909.30
export node_ip=192.168.122.21
 - id, pass for hub.juniper.net is needed

cat > command_servers.yml << EOF
---
command_servers:
    server1:
        ip: ${node_ip}
        connection: ssh
        ssh_user: root
        ssh_pass: root
        sudo_pass: root
        ntpserver: 0.centos.pool.ntp.org

        registry_insecure: false
        container_registry: ${docker_registry}
        container_tag: "${container_tag}"
        config_dir: /etc/contrail

        contrail_config:
            database:
                type: postgres
                dialect: postgres
                password: contrail123
            keystone:
                assignment:
                    data:
                      users:
                        admin:
                          password: contrail123
            insecure: true
            client:
              password: contrail123
            auth_type: basic-auth
EOF


cat > install-cc.sh << EOF
docker login hub.juniper.net
docker pull ${docker_registry}/contrail-command-deployer:${container_tag}
docker run -td --net host -e orchestrator=none -e action=import_cluster -v /root/command_servers.yml:/command_servers.yml -v /root/contrail-ansible-deployer/config/instances.yaml:/instances.yml --privileged --name contrail_command_deployer ${docker_registry}/contrail-command-deployer:${container_tag}
EOF

bash install-cc.sh
```

After few minutes, https://(ip):9091 will be used to access Command webui.

If something won't work, this command will show installation log.
```
# docker logs -f contrail_command_deployer

To retry, type
# docker rm -f contrail_command contrail_psql contrail_command_deployer
and type this again
# bash install-cc.sh
```


## Contrail Fabric Automation

Contrail Command has 'Fabric' tab to automate the configuration of Junos Series, currently MX, QFX, SRX is the target.

This youtube video covers many features of fabric automation, such as EVPN / VXLAN fabric cration, and CRB concept in this setup.
 - https://www.youtube.com/watch?v=j1NuZzHep54

Main theme of this feature is to automate EVPN / VXLAN setting, which can be cumbersome when number of virtual-networks are large, and it will be convienient method to extend vrouters' virtual-network to baremetal world.

Adding to the features in youtube, recent version of Fabric automation also supports those features.
 - Edge Routing / Bridging: https://www.juniper.net/documentation/en_US/release-independent/solutions/topics/task/configuration/edge-routed-overlay-cloud-dc-configuring.html
 - Hitless upgrade
 - PNF integration

Note: This site has good amount of info about r5.1 fabric feature
 - https://github.com/tonyliu0592/contrail/wiki/Solution-Guide-Contrail-Fabric-Management

So if you need some EVPN / VXLAN fabric which can work with Tungsten Fabric, those switches and Command UI will be one choise.

Note: If something won't work well, please check these logs and files
```
# docker logs -f --tail=10 config_devicemgr_1 ## ansible logs
# docker exec -it config_devicemgr_1 bash
  # cd /opt/contrail/fabric_ansible_playbooks/config ## abstract config for each device, which is used in each jinja template
```

### vQFX setting

Let me share vQFX setting which I'm using currently.

As a hypervisor, I currently use Centos7, which can run well with vQFX 18.4R1, which can be downloaded from juniper site.
https://www.juniper.net/us/en/dm/free-vqfx-trial/

With that, those virt-install command could create vQFX-re and vQFX-pfe.
```
name=vqfx183-re
virt-install \
  --name ${name} \
  --ram 1024 \
  --vcpus 1 \
  --import \
  --disk /root/vqfx/${name}.img,bus=ide,format=raw \
  --network network=default,model=e1000 \ ## em0
  --network network=br_int_183,model=e1000 \ ## internal (connected to pfe)
  --network network=br_int_183,model=e1000 \ ## internal
  --network network=181_183,model=e1000 \ ## xe-0/0/0
  --network network=182_183,model=e1000 \ ## xe-0/0/1
  --network network=183_bms1,model=e1000 \ ## xe-0/0/2
  --graphics none

name=vqfx183-pfe
virt-install \
  --name ${name} \
  --ram 2048 \
  --vcpus 1 \
  --import \
  --disk /root/vqfx/${name}.img \
  --network network=default,model=e1000 \ ## em0
  --network network=br_int_183,model=e1000 \ ## internal (connected to re)
  --graphics none
```

To connect each vQFX, linux bridge based on libvirt net is used (that can be configured from virt-manager)

```
# virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 181_183              active     yes           yes
 181_184              active     yes           yes
 182_183              active     yes           yes
 182_184              active     yes           yes
 183_bms1             active     yes           yes
 184_bms2             active     yes           yes
 br_int_181           active     yes           yes
 br_int_182           active     yes           yes
 br_int_183           active     yes           yes
 br_int_184           active     yes           yes


# EDITOR=/bin/cat virsh net-edit 181_183
<network>
  <name>181_183</name>
  <uuid>b87b51ca-f9c7-4f67-8963-cc267551a5bb</uuid>
  <bridge name='virbr5' stp='on' delay='0'/>
  <mac address='52:54:00:22:d4:d6'/>
  <domain name='181_183'/>
</network>
Network 181_183 XML configuration not changed.
```


I currently use 4 vQFXes, with 2 leaf, 2 spine configuration
 - typically, spine will get crb gateway, route reflector and leaf will get crb access when crb is used, and spine will get route reflector, and leaf will get erb-ucast-access when erb is used

Since lldp is needed for topology discovery to work well, those command also will be needed to update linux bridge setting.
 - https://interestingtraffic.nl/2017/11/21/an-oddly-specific-post-about-group_fwd_mask/
```
for i in $(ls /sys/class/net/virbr*/bridge/group_fwd_mask); do echo 16384 > $i; done
```

Then, 4 vQFXes with lldp neighbor configured is available, to test most of functionalites of fabric automation.


### LACP against linux bridge

Fabric Automation has a feature to implement ESI-LAG, which supplies l2 high availability between nodes without the need for MC-LAG.

To use this feature, virtual-port-group can be created with two interfaces assigned.
 - it will create aggregated-ethernet among two (or more than three) vQFXes: https://github.com/Juniper/contrail-controller/blob/master/src/config/fabric-ansible/ansible-playbooks/roles/cfg_overlay_multi_homing/templates/juniper_junos-qfx_overlay_multi_homing.j2

Having said that, since linux bridge won't allow LACP packet to go through, it is not possible to try this feature based on kvm environment.

To overcome this, one possible configuration is to modify 'bridge' module to allow LACP packets to go through them.
 - https://lists.linuxfoundation.org/pipermail/bridge/2010-January/006944.html
 - https://wiki.centos.org/HowTos/BuildingKernelModules

```
For centos7, this patch can be used:

--- linux-3.10.0-1062.el7/net/bridge/br_input.c.20191009	2019-07-19 04:58:03.000000000 +0900
+++ linux-3.10.0-1062.el7/net/bridge/br_input.c	2019-10-09 21:06:34.310945313 +0900
@@ -302,6 +302,9 @@
 		case 0x01:	/* IEEE MAC (Pause) */
 			goto drop;
 
+		case 0x02:	/* LACP */
+			goto forward;
+
 		case 0x0E:	/* 802.1AB LLDP */
 			fwd_mask |= p->br->group_fwd_mask;
 			if (fwd_mask & (1u << dest[5]))


or this if it needs to be enabled / disabled per bridge (echo 16388 > /sys/class/net/virbr0/bridge/group_fwd_mask, although I haven't yet tried this patch):

--- ./linux-3.10.0-1062.el7/net/bridge/br_sysfs_br.c.20191009	2019-07-19 04:58:03.000000000 +0900
+++ ./linux-3.10.0-1062.el7/net/bridge/br_sysfs_br.c	2019-10-09 01:24:27.183776062 +0900
@@ -297,7 +297,7 @@
 		return -EINVAL;
 
 	if (new_addr[5] == 1 ||		/* 802.3x Pause address */
-	    new_addr[5] == 2 ||		/* 802.3ad Slow protocols */
+	    /* new_addr[5] == 2 ||*/		/* 802.3ad Slow protocols */
 	    new_addr[5] == 3)		/* 802.1X PAE address */
 		return -EINVAL; 
```

Building bridge.ko and replacing bridge.ko file under /lib/modules (OS reboot is required), LACP packet should be transparently go through linux bridge.

After that, ESI-LAG can be tried with two leaf vQFXes and two linux bridges for each vQFX port, and one linux VM.
 - When I tried vQFX port shutdown (set interfaces xe-0/0/2 disable) with LACP fast, failover completes in 3 seconds.
```
When VyOS is used as the linux VM, this configuration can be used:
# set interfaces bonding bond0
# set interfaces ethernet eth0 bond-group bond0
# set interfaces ethernet eth1 bond-group bond0

Note: by default, only lacp slow could be configured. When bond0 is in 'disabled' state, it can be updated by /sys.
# set interfaces bonding bond0 disable
# commit

$ sudo su -
# echo 1 > /sys/class/net/bond0/bonding/lacp_rate
```

One additonal note, when linux bonding is used with virtio-net, it can't send LACP. For this purpose, e1000 can be used instead.
 - https://lists.linuxfoundation.org/pipermail/virtualization/2016-January/031520.html

### vQFX limitation

With this lab setup, most of the fabric-automation features can be covered with a single lab server.

Having said that, there are currently two known limitations, so let me describe about them.

> 1. BMS to VM l2 vxlan won't work well

Since the latest available vQFX version is 18.4R1, it has known issue with the arp response over vxlan becoming vni 0, in some situation. (This issue is already fixed in 18.4R2 and later) 

```
# tcpdump -nn -i vnet148 udp and port 4789
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vnet148, link-type EN10MB (Ethernet), capture size 262144 bytes
01:25:26.746557 IP 192.168.122.187.59217 > 172.16.15.249.4789: VXLAN, flags [I] (0x08), vni 6
ARP, Request who-has 10.0.1.202 tell 10.0.1.5, length 28
01:25:26.848000 IP 172.16.15.249.11020 > 192.168.122.187.4789: VXLAN, flags [I] (0x08), vni *0*
ARP, Reply 10.0.1.202 is-at 52:54:00:65:94:f4, length 46

(vni 0 from vQFX should be vni 6)
```

So if this specific feature won't work well, please add temporarily static arp to the VMs on vRouters.
```
# arp -s 10.x.x.x xx:xx:xx:xx:xx:xx
```


> 2. LLDP based underlay setting won't work well

Since vQFX's > show chassis mac-addresses and > show lldp local-information show different value, LLDP based underlay setting won't work well when vQFX is used. (When combined with physical QFX, it works well)
So please set underlay routing manually when vQFX is used.


### Integration with fabric automation and vRouters

Since vRouter uses vxlan internally, ironically, it needs some trick to put them under QFXes which is configured by fabric automation, since by default it will configure overlay vni for each virtual-port-group.

To workaround this, esi-lag without overlay can be used for Tungsten Fabric nodes, such as control, vRouter, and contrail-command itself.
https://www.juniper.net/documentation/en_US/junos/topics/example/example-evpn-vxlan-lacp-configuring-qfx-series.html

To try this, you can firstly set up management switch, and connect all the Tungsten Fabric nodes to them and install all the nodes, before data plane access is not availble.
 - Note: when Tripleo is used, you might need to disable ping check to make installation work
 - When config-database needs HA, some manual config might needed before fabric automation is installed

After that, you can configure 'underlay' VN in contrail command, with subnets which can be assigned to tungsten fabric nodes, and do brownfield onboard, and configure virtual-port-group on the QFX ports, which has tungsten fabric nodes connected.
 - Note: to make that work, please don't connect logical-router to underaly VN, since in that case, irb will be inside a VRF, and it can't contact to the underlay subnet

After that, each tungsten fabric nodes can ping to vQFX loopback ips, which is set by contrail fabric autmation, and contrail fabric are fully UP.
 - To reach loopback ip from tungsten fabric nodes, some underlay routing protocol might be needed


As an example, I tried two spines and 8 leaves setup, and set them up with (mostly) fabric automation features.
![8-leaves](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/vRouter-ESI-LAG-BMS-diagram.png)

Detailed configuration and ping results are attached.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/8-leaves-vqfx-config.txt
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/8-leaves-contrail-config.txt
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/8-leaves-ping-result.txt

Note:
Some configuration such as irb for vRouters, and bgp export policy for EVPN route needs to be written manually.


### PNF integration

Let me briefly describe the several approach of PNF integration with contrail networking.

Although Contrail Command has its own PNF integration based on service-appliance-set in Tungsten Fabric, it can also be configured based on logical-route and virtual-port-group.

Since latest fabric automation supports virtual-port-group and logical-router to setup PNF access to virtual-network, those setting also could be possible.
- vm1 (belong to vn1, and get floating-ip from public-vn) - vn1 - public-vn (external is checked, and connected to lr1) - lr1 (vxlan-routing, extended to qfx, connected to vn1, vn-pnf-left) - vn-pnf-left - pnf1 - vn-pnf-right - lr2 (connected to vn-pnf-right, extended to qfx) - border-router

From vRouter side, it just means lr1 is extended to QFX and can send traffic to the underlay through this, and from fabric automation side, it just connected lr1 and lr2 to pnf1 and lr2 to border-leaf, based on some routing protocol (ospf or bgp might be a choice, both pnf and QFX VRFs need to be manually configured).

So with contrail command, vRouter's vm can be integrated with PNF, although some manual config is still needed.

### Ironic integration

### Appformix integration

Since most of the analytics features of contrail-command is implemented in appformix module, installation of that module is also needed to enable visualization features.

To do this, most straightforward way is to use provision_cluster option of contrail-command-deployer.
 - It installs contrail command and contrail cluster with instances.yaml simultaneously

Let me firstly begin with the minimal setup with one node for contrail-command, contrail-controller, appformix.

```
# At mininum, Mem 16GB, Disk: 50GB is required

export docker_registry=hub.juniper.net/contrail
export container_tag=1910.23
export node_ip=192.168.122.96
export hub_username=xxx
export hub_password=yyy
 - id, pass for hub.juniper.net is needed

cat > install-cc.sh << EOF
systemctl stop firewalld; systemctl disable firewalld
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce-18.03.1.ce
systemctl start docker
docker login ${docker_registry}
docker pull ${docker_registry}/contrail-command-deployer:${container_tag}
docker run -td --net host -e orchestrator=openstack -e action=provision_cluster -v /root/command_servers.yml:/command_servers.yml -v /root/instances.yaml:/instances.yml --privileged --name contrail_command_deployer ${docker_registry}/contrail-command-deployer:${container_tag}
EOF


cat > command_servers.yml << EOF
---
command_servers:
    server1:
        ip: ${node_ip}
        connection: ssh
        ssh_user: root
        ssh_pass: root
        sudo_pass: root
        ntpserver: 0.centos.pool.ntp.org

        registry_insecure: false
        container_registry: ${docker_registry}
        container_tag: ${container_tag}
        config_dir: /etc/contrail

        contrail_config:
            database:
                type: postgres
                dialect: postgres
                password: contrail123
            keystone:
                assignment:
                    data:
                      users:
                        admin:
                          password: contrail123
            insecure: true
            client:
              password: contrail123
user_command_volumes:
  - /opt/software/appformix:/opt/software/appformix
EOF


cat > instances.yaml << EOF
global_configuration:
  CONTAINER_REGISTRY: ${docker_registry}
  REGISTRY_PRIVATE_INSECURE: false
  CONTAINER_REGISTRY_USERNAME: ${hub_username}
  CONTAINER_REGISTRY_PASSWORD: ${hub_password}
provider_config:
  bms:
    ssh_user: root
    ssh_pwd: root
    ntpserver: 0.centos.pool.ntp.org
    domainsuffix: local
instances:
  bms1:
    ip: ${node_ip}
    ssh_user: root
    ssh_pwd: root
    provider: bms
    roles:
      config:
      config_database:
      control:
      webui:
      analytics:
      analytics_database:
      analytics_snmp:
      openstack_control:
      openstack_network:
      openstack_storage:
      openstack_monitoring:
      appformix_openstack_controller:
      appformix_controller:
      appformix_compute:
      openstack_compute:
contrail_configuration:
  CLOUD_ORCHESTRATOR: openstack
  RABBITMQ_NODE_PORT: 5673
  ENCAP_PRIORITY: VXLAN,MPLSoGRE,MPLSoUDP
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
  CONTRAIL_CONTAINER_TAG: "${container_tag}"
  JVM_EXTRA_OPTS: -Xms128m -Xmx1g
  COLLECTOR_PORT: 18086
  ANALYTICS_API_INTROSPECT_LISTEN_PORT: 18090
kolla_config:
  kolla_globals:
    enable_haproxy: no
    enable_ironic: no
    enable_cinder: no
    enable_heat: no
    enable_glance: no
    enable_barbican: no
  kolla_passwords:
    keystone_admin_password: contrail123
appformix_configuration:
    appformix_license:  /opt/software/appformix/appformix-openstack-3.1.sig
EOF

bash install-cc.sh ## it takes about 60 minutes to finish installation


Note: before typing bash install-cc.sh, please upload those files to /opt/software/appformix
 appformix-3.1.x.tar.gz
 appformix-dependencies-images-3.1.x.tar.gz
 appformix-network_device-images-3.1.x.tar.gz
 appformix-openstack-images-3.1.x.tar.gz
 appformix-platform-images-3.1.x.tar.gz
 appformix-openstack-3.1.sig


To see the installation status, those two commands can be used
# docker logs -f contrail_command_deployer ## for command installation
# tail -f /var/log/contrail/deploy.log     ## for contrail cluster installation
```

After that, contrail-command should show some additional views such as 'Topology View', 'Overlay / Underlay correlation', in a similar sense with contrail-webui.
![TopologyView](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/CommandTopologyView.png)

Note: Since analytics-api introspect port is changed from 8090 to 18090, contrail-status won't work well in this setup.
To workaround this, please try this procedure.
```
1. create contrail-status container to be modified
 # vi /usr/bin/contrail-status
 remove '--rm' from docker run option
 # contrail-status
2. modify port definition file
 # docker cp contrail-status:/usr/lib/python2.7/site-packages/sandesh_common/vns/constants.py .
 # sed -i 's/8090/18090/' constants.py
 # docker cp constants.py contrail-status:/usr/lib/python2.7/site-packages/sandesh_common/vns/constants.py
3. replace docker run command by docker start command
 # vi /usr/bin/contrail-status
 replace docker run command by this: docker start -a contrail-status
X. type contrail-staus and check if it works as expected
 # contrail-status 
```

### contrail-vcenter-fabric-manager

## Contrail Healthbot

## Contrail multicloud

### container deployment
### baremetal instance deployment

## Contrail SD-WAN
