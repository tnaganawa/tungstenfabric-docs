
Table of Contents
=================

   * [Contrail Command](#contrail-command)
   * [Contrail Fabric Automation](#contrail-fabric-automation)
      * [vQFX setting](#vqfx-setting)
      * [LACP against linux bridge](#lacp-against-linux-bridge)
      * [vQFX limitation](#vqfx-limitation)
      * [Integration with fabric automation and vRouters](#integration-with-fabric-automation-and-vrouters)
      * [Multi fabric setup](#multi-fabric-setup)
      * [PNF integration and External Access](#pnf-integration-and-external-access)
      * [Ironic integration](#ironic-integration-wip)
   * [Contrail Healthbot](#contrail-healthbot)
   * [Contrail multicloud](#contrail-multicloud)
      * [container deployment](#container-deployment)
      * [baremetal instance deployment](#baremetal-instance-deployment)
   * [Contrail Insights](#contrail-insights)
   * [Contrail SD-WAN](#contrail-sd-wan)
   * [Troubleshooting Tips](#troubleshooting-tips)


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

> 3. BMS to VM l2 vxlan won't work well, since vRouter won't do arp proxy for EVPN BMS mac

For 18.4R2 and later (including 18.4R2-S2), vQFX won't respond to arp from other EVPN peers, since it should be handled by arp proxy of its peers.
Having said that, vRouter currently won't do arp proxy in that case ..
 - This behaivor might be changed in future release

To make this work, 
```
set vlans vlan-name no-arp-suppression
```
need to be added to each bridge-domain.

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

### Multi fabric setup

In most cases, multi fabric is a requirement to implement large DC setup, and fabric-manager supports that in several ways.

One way is to use L3DCI feature which it natively supports, to configure EVPN T5 peer between border QFX.

Having said that, it is also possible to configure l2 traffic between multi fabrics, with some manual configuration on QFX and tungsten fabric config-database.

Overall idea is to set up EVPN peers between RRs in both fabrics, and make each leaf switch send EVPN routes between them.

#### Implementation

I'll start with this minimal setup (1912.32 is used).
![multi-l2-fabric](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/multi-l2-fabric-diagram.png)
 
Both fabric1 and fabric2 has one spine and one leaf, and spines have l3 connectivity between them.
 - super-spine could be between them, if they don't join EVPN peer

Firstly, greenfield onboarding or brownfield onboarding is done for two fabrics.
 - Route Reflector for spine, and ERB-Unicast-gateway for leaf are configured
 - underlay bgp inside each fabric will be automatically configured if greenfield onboarding is used

After that, spine's interfaces are manually configured ip, and unicast bgp between them.
```
vqfx191:
 set interfaces xe-0/0/4 unit 0 family inet address 172.16.12.129/30
 set protocols bgp group IPCLOS_eBGP neighbor 172.16.12.130 peer-as 65201
vqfx193:
 set interfaces xe-0/0/3 unit 0 family inet address 172.16.12.130/30
 set protocols bgp group IPCLOS_eBGP neighbor 172.16.12.129 peer-as 65300
```

Finally, I configured Clusters > Advanced Options > bgp-router, and updated associated peer, to include other fabric's spine.
 - If one of the peers is configured, schema-transformer automatically update other side

Then EVPN peer will be up between RRs to make all the leaves interchange EVPN route,
and other objects like virtual-network, virtual-port-group, logical-router can be used in both fabric.
 - RR and tungsten fabric control EVPN peer also will be up

I'll attach BMS to BMS and BMS to VM ping result for L2, L3 traffic, for reference purpose.
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/multi-l2-fabric-vQFX-config.txt
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/multi-l2-fabric-ping-result.txt

So it is not too difficult to configure multiple fabrics and vRouters under that, even if l2 extension between fabrics is a requirement.

### PNF integration and External Access

Let me describe the several approach of PNF integration with contrail networking.

Although Contrail Command has its own PNF integration based on service-appliance-set in Tungsten Fabric, it can also be configured based on logical-route and virtual-port-group.

Since fabric automation supports virtual-port-group and logical-router to setup PNF access to virtual-network, those setting also could be possible.

From R2003, it begins to support unicast BGP to external router and unmanaged PNF and allows similar setup without manual config.
 - https://www.juniper.net/documentation/en_US/contrail20/topics/topic-map/connect-third-party-device-cc.html

So .., this allows several setup for firewall and external routers.
 - https://www.juniper.net/documentation/en_US/day-one-books/TW_DCDeployment.v2.pdf
 - https://www.juniper.net/documentation/en_US/release-independent/solutions/topics/topic-map/service-chaining.html

Since CRB Spine or ERB leaf also can do l3 routing, there are several configuration option. Each has its pros and cons, so it can be chosen based on your environment.
 1. all the l3 rouitng are done in fabric device, and connect that to external router with unicast BGP
  - 1-1. same as 1, but create two VRFs for North-South firewall
 2. with two VRFs for l3 routing with parts of l2 vni connected to each, and firewall device between them, for East-West firewall
 3. all l3 routing will be done by external router (bridged overlay): one way to configure this is to create VRF only on MX, with MX as ERB leaf
 4. all l3 routing will be done by firewall (firewall for all l3): one way to configure this is to assign a logical-router to  L2VNI, and connect each logical-router with PNF with vlan tag

Since vRouter can share VXLAN L3VRF with fabric device, each option can be combined with vRouter.

Note:
One more possible options is to add vxlan VNF service-chain, instead of PNF.
It can be used with 1-1, 2 scenario, although I haven't yet seen it can work with 4, since it requires vlan subinterface ..


### Ironic integration WIP
Contrail has some feature to integrate with ironic, based mainly on fabric-manager.

To use this feature, several things need to be taken care of.

Firstly, since ironic's workflow use 'ironic-provision' VN to clean volume and install new image data, this VN need to access the ironic-conductor, which is located at the underlay.
So this VN need to be configured with both overlay and underlay access, and the most straight forward way is to use similar setting with vRouter integration chapter.

Secondly, since ironic itself won't have much knowledge about how to configure switch port, some info about QFX port and BMS need to be configured in both databases (ironic and contrail configdb).
For this purpose, contrail command has some workflow to firstly fill contrail configdb, and fill ironic db with that info.

#### ironic setup

##### Underlay setup

Firstly, you need to create two leaves setting with fabric-manager.
 - I used vqfx194 to connect AIO node of contrail, and vqfx195 to connect BMS (two lean spines vqfx191, vqfx192 are connected to them)

Since CSN feature is needed to manage BMS, AIO has vrouter role with TSN_EVPN_MODE specified
```
provider_config:
  bms:
   ssh_user: root
   ssh_pwd: root
   ssh_public_key: /root/.ssh/id_rsa.pub
   ssh_private_key: /root/.ssh/id_rsa
   domainsuffix: local
   ntpserver: 0.centos.pool.ntp.org
instances:
  bms1:
    provider: bms
    ip: 192.168.122.97
    roles:
      config_database:
      config:
      control:
      analytics:
      analytics_database:
      webui:
      openstack:
      vrouter:
        TSN_EVPN_MODE: true
        VROUTER_GATEWAY: 172.18.0.1
      openstack_compute:
contrail_configuration:
  CLOUD_ORCHESTRATOR: openstack
  CONTROL_NODES: 172.18.0.97
  TSN_NODES: 172.18.0.97 ## set the same IP with vhost0
  CONTRAIL_CONTAINER_TAG: 1911.31
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
  JVM_EXTRA_OPTS: "-Xms128m -Xmx1g"
kolla_config:
  kolla_globals:
    enable_haproxy: no
  kolla_passwords:
    keystone_admin_password: contrail123
global_configuration:
  CONTAINER_REGISTRY: hub.juniper.net/contrail
  CONTAINER_REGISTRY_USERNAME: username
  CONTAINER_REGISTRY_PASSWORD: password
```

After AIO is connected to fabric and fabric is created by that, 'ironic-provision' VN will be created from contrail-command, and it is extended to leaf QFX for BMS access (vqfx195 in this setup) to configure irb to reach underlay subnet.
![ironic-provision-VN](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/ironic-ironic-provision-VN.png)

To use AIO CSN for the dhcp access from this leaf, Fabric > fabric-name > physical router name > edit, Associated Service Nodes need to be filled
 - without this definition, vqfx195 won't receive EVPN T2 route from contrail-controller
![associate-CSN](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/ironic-associated-csn.png)

##### BMS setup 

After that, you firstly need to create flavor and image which will be used by ironic and nova. 
You can use command's UI to do this: Workloads > Flavors and Workloads > Images (please set Server Type: Baremetal Server)

For this setup, I used those five images.
 - curl -O http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
 - curl -O http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-initramfs
 - curl -O http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-kernel
 - curl -O https://tarballs.openstack.org/ironic-python-agent/coreos/files/coreos_production_pxe-stable-queens.vmlinuz
 - curl -O https://tarballs.openstack.org/ironic-python-agent/coreos/files/coreos_production_pxe_image-oem-stable-queens.cpio.gz

Those openstack cli also can be used, but please set --property server_type=baremetal, since without that, it cannot be used from command webui.

```
$ glance image-create --name my-kernel --visibility public --property server_type='baremetal' \
  --disk-format aki --container-format aki < cirros-0.4.0-x86_64-kernel
$ MY_VMLINUZ_UUID=$(openstack image show -c id my-kernel -f value)
$ glance image-create --name my-image.initrd --visibility public --property server_type='baremetal' \
  --disk-format ari --container-format ari < cirros-0.4.0-x86_64-initramfs
$ MY_INITRD_UUID=$(openstack image show -c id my-image.initrd -f value) 
$ glance image-create --name my-image --visibility public \
  --disk-format qcow2 --container-format bare --property server_type='baremetal' --property \
  kernel_id=$MY_VMLINUZ_UUID --property \
  ramdisk_id=$MY_INITRD_UUID < cirros-0.4.0-x86_64-disk.img
$ glance image-create --name deploy-vmlinuz --visibility public --property server_type='baremetal' \
      --disk-format aki --container-format aki < coreos_production_pxe.vmlinuz
$ glance image-create --name deploy-initrd --visibility public --property server_type='baremetal' \
      --disk-format ari --container-format ari < coreos_production_pxe_image-oem.cpio.gz

$ openstack flavor create --id auto --ram 512 --vcpus 1 --disk 4 --property baremetal=true --public baremetal
```


Then BMS's ipmi port will be filled in with some info about QFX port, which BMS is connected, Infrastructure -> Servers -> Create -> Detailed -> Baremetal
 - please set enable PXE, and fill in IPMI info
 - This information is firstly saved in various tables in contrail configdb, such as node, port, virtual-machine-interface 
![ironic-bms-port-info](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/ironic-bms-port-info.png)


After that, you can enroll this BMS definition in ironic db with Infrastructure -> Cluster -> Cluster Nodes -> Baremetal Servers -> Add -> Baremetal Servers

Then you can see ironic db is filled in with this info.
```
$ openstack baremetal node list
+--------------------------------------+----------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name     | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+----------+---------------+-------------+--------------------+-------------+
| c3404f1c-ea67-481c-a733-58ba86527d71 | 195_bms2 | None          | power on    | manageable         | False       |
+--------------------------------------+----------+---------------+-------------+--------------------+-------------+

$ openstack baremetal node provide c3404f1c-ea67-481c-a733-58ba86527d71
$ openstack baremetal node list
+--------------------------------------+----------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name     | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+----------+---------------+-------------+--------------------+-------------+
| c3404f1c-ea67-481c-a733-58ba86527d71 | 195_bms2 | None          | power on    | available          | False       |
+--------------------------------------+----------+---------------+-------------+--------------------+-------------+
$
```

##### Create BMS Instance

When baremetal node status become 'available', it can be used to instantiate BMS node from horizon and command UI, in a similar manner to create VMs. 
 - for availability zone, nova-baremetal need to be specified

When everything is working well, CSN node will get a vif for BMS, and return dhcp response when BMS send dhcp request.
 - ironic-provision subnet's ip will be assigned firstlly
```
# ./contrail-introspect-cli/ist.py vr intf 16
+-------+--------------------------------------+--------+-------------------+-------------+---------------+---------+--------------------------------------+
| index | name                                 | active | mac_addr          | ip_addr     | mdata_ip_addr | vm_name | vn_name                             |
+-------+--------------------------------------+--------+-------------------+-------------+---------------+---------+--------------------------------------+
| 16    | 5eb8e7cc-485a-47cc-8756-89355432492a | Active | 52:54:00:95:75:59 | 172.18.11.3 | 0.0.0.0       | n/a     | default-domain:admin:ironic-         |
|       |                                      |        |                   |             |               |         | provision                           |
+-------+--------------------------------------+--------+-------------------+-------------+---------------+---------+--------------------------------------+
#
```

Then it will receive IP by dhcp, do tftp to get kernel and boot, and do some tasks with iSCSI.
 - Having said that, vQFX 18.4R1 has several limitations, which I haven't yet fully worked around (and this doc will be WIP status for a while :( )
 - 1. when l2 packets is sent, it sometimes send VNI 0 packets, and it can't receive IP through dhcp 
 - 2. tftp will be pretty slow, since vQFX has pretty high latency (- 100 ms)
 - 3. Since vQFX can't forward over 1,500 bytes packets, iSCSI won't work well


### contrail-vcenter-fabric-manager

This feature is to sync dv-portgroup info in vCenter to VN/VPG info of fabric-manager.
 - https://github.com/tungstenfabric/tf-specs/blob/master/contrail-vcenter-fabric-manager.md

To fascilitate this, lldp need to be enabled on ESXi and ToR, to identify physical-interfaces which is connected to each ESXi.

When dv-portgroup is created, it will be synced to config-database, to create virtual-network with the name with dv-portgroup (uuid is the same as the one for dv-portgroup in vCenter database)

When some VM is created on an ESXi, VPG will be automatically created on the physical-interface which that ESXi is connected.

Since it only creates L2 access from fabric-manager perspective, it can be connected to logical-routers, to supply direct l3 routing in QFX, or some l3 routing through PNF.

## Contrail Healthbot

Currently, appformix doesn't support routing protocol monitoring, route table size monitoring and so on.

Although it is not fully integrated, one of the most natural choise is to install healthbot as its companion.
 - https://www.juniper.net/us/en/products-services/sdn/contrail/contrail-healthbot/

Since it natively supports EVPN/VXLAN monitoring with per device-group heatmap view, it is not a difficult task to implement this feature, with several fabircs implemented by fabric-manager.
 - https://www.youtube.com/watch?v=B20qjChoKfg

To set up this, those guides can be followed.
 - https://www.juniper.net/documentation/en_US/healthbot/topics/concept/healthbot-getting-started.html

```
Practically, those two commands are all that are needed.
# yum -y install healthbot-x.x.x.noarch.rpm
# healthbot setup
# healthbot start

Then, https://healthbot-node-ip:8080 will give healthbot webui.
```

## Contrail multicloud

Some usecases is described in this channel.
 - https://www.youtube.com/channel/UCXUny7HKBdyakn3-UOdkhsw
 
### container deployment
### baremetal instance deployment

## Contrail Insights

Since most of the analytics features of contrail-command is implemented in contrail-insights module (appformix / appformix-flows internally), installation of that module is also needed to enable visualization features.

To do this, most straightforward way is to use provision_cluster option of contrail-command-deployer.
 - It installs contrail command and contrail cluster with instances.yaml simultaneously

Let me firstly begin with the minimal setup with two nodes for contrail-command, contrail-controller, appformix, appformix-flows.

```
# At mininum, Mem 16GB, Disk: 50GB is required

export docker_registry=hub.juniper.net/contrail
export container_tag=2005.62
export node_ip=192.168.122.252
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
  controller1:
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
      appformix_compute:
      openstack_compute:
  insights1:  ### from R2005, this should be the same as the hostname of this node
    provider: bms
    ip: 192.168.122.251
    roles:
      appformix_controller:
      appformix_bare_host:
      appformix_flows:

contrail_configuration:
  CLOUD_ORCHESTRATOR: openstack
  RABBITMQ_NODE_PORT: 5673
  ENCAP_PRIORITY: VXLAN,MPLSoGRE,MPLSoUDP
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
  CONTRAIL_CONTAINER_TAG: "${container_tag}"
  JVM_EXTRA_OPTS: -Xms128m -Xmx1g
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
xflow_configuration:
  clickhouse_retention_period_secs: 7200
  loadbalancer_collector_vip: 192.168.122.250
EOF

bash install-cc.sh ## it takes about 60 minutes to finish installation


To see the installation status, those two commands can be used
# docker logs -f contrail_command_deployer ## for command installation
# tail -f /var/log/contrail/deploy.log     ## for contrail cluster installation
# docker exec -t contrail-player_xxxxx tail -f /var/log/ansible.log ## to see detailed ansible-player log during contrail cluster installation
```

After that, contrail-command should show some additional views such as 'Topology View', 'Overlay / Underlay correlation', in a similar sense with contrail-webui.
![TopologyView](https://github.com/tnaganawa/tungstenfabric-docs/blob/master/CommandTopologyView.png)



## Contrail SD-WAN

## Troubleshooting Tips
### login to command failed for some reason
1. Command internally uses 'local storage' (not only cookie) for login auth, so for some situation, it also needs to be cleared.
 - Firefox: Web Developer > Storage Inspector > Local Storage > Delete All
### fabric automation won't push correct configuration
1. It internally expects that the first one of ENCAP_PRIORITY is VXLAN, so it needs to be set to have correct configuration.
