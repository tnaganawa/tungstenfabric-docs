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
export container_tag=1907.55
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
# docker stop contrail_command contrail_psql
# docker rm contrail_command contrail_psql contrail_command_deployer
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

So if you need some EVPN / VXLAN fabric which can work with Tungsten Fabric, those switches and Command UI will the one choise.

### vQFX setting

Let me share vQFX setting which I'm using currently.

As a hypervisor, I currently use Centos7, which can run well with vQFX 18.4R1, which can be downloaded from juniper site.
https://www.juniper.net/us/en/dm/free-vqfx-trial/

With that, those virt-install command could create vQFX-re and vQFX-pfe.

To connect each vQFX, linux bridge based on libvirt net is used (that can be configured from virt-manager)

virsh net-list
EDITOR=/bin/cat virsh net-edit 181_183


I currently use 4 vQFXes, with 2 leaf, 2 spine configuration
 - typically, spine will get crb gateway, route reflector and leaf will get crb access when crb is used, and spine will get route reflector, and leaf will get erb-ucast-access when erb is used

Since lldp is needed for topology discovery to work well, those command also will be needed to update linux bridge setting.

Then, 4 vQFXes with lldp neighbor configured is available, to test most of functionalites of fabric automation
 - Note: having said that, since linux bridge won't allow LACP to pass through, EVPN multihoming can't be tried with this setup

### Integration with fabric automation and vRouters

Since vRouter use vxlan internally, ironically, it can't be under QFXes which is configured by fabric automation :(, at least if inter-vni routing is set by logical-routers.

To workaround this, esi-lag without overlay can be used for Tungsten Fabric nodes, such as control, vRouter, and contrail-command itself.
https://www.juniper.net/documentation/en_US/junos/topics/example/example-evpn-vxlan-lacp-configuring-qfx-series.html

To configure this, you can firstly configure management switch, and connect all the Tungsten Fabric nodes, to them and install all the nodes, before data plane access is not availble.
 - Note: when Tripleo is used, you might need to disable ping check to make installation work

After that, you can configure 'underlay' vn in contrail command, with subnets which can be assigned tungsten fabric nodes, and do brownfield onboard, and configure virtual-port-group on the QFX ports, which has tungsten fabric nodes connected.
 - Note: to make that work, please don't connect logical-router to underaly vn
 - this procedure need to be verified if it works well with latest module

After that, each tungsten fabric nodes can ping with vQFX loopback ips, which is set by contrail fabric autmation, and contrail fabric are fully UP!


### PNF integration

Let me briefly describe the serveral approach of PNF integration with contrail networking.

Although Contrail Command has its own PNF integration based on service-appliance-set in Tungsten Fabric, it can also be configured based on logical-route and virtual-port-group.

Since latest fabric automation supports virtual-port-group and logical-router to setup PNF access to virtual-network, those setting also could be possible.
- vm1 (belong to vn1, and get floating-ip from public-vn) - vn1 - public-vn (external is checked, and connected to lr1) - lr1 (vxlan-routing, extended to qfx, connected to vn1, vn-pnf-left) - vn-pnf-left - pnf1 - vn-pnf-right - lr2 (connected to vn-pnf-right, extended to qfx) - border-router

From vRouter side, it just means lr1 is extended to QFX and can send traffic to the underlay through this, and from fabric automation side, it just connected lr1 and lr2 to pnf1 and lr2 to border-leaf, based on some routing protocol (ospf or bgp might be a choice, both pnf and QFX VRFs need to be manually configured).

So with contrail command, vRouter's vm can be integrated with PNF, although some manual config is still needed.

## Contrail multicloud

## Contrail SD-WAN
