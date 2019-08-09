OpenWRT Config
=========

Ansible role to aid in the configuration of OpenWRT.

Requirements
------------

SSH access to the router.

Ansible _usually_ needs python installed on the target host (router, in this case), but leverage of the _raw_ module allowed to skip this requirement.

Description
-----------

Because of reasons(tm), I needed multiple routers (on VMs) to route multiple "client" networks into a common "backbone" network. A third network was to be set aside for "management". OpenWRT was selected for the application. Each interface belongs to a group and they form a zone. Traffic will be then allowed to the whole zone.

Due to the number of networks (+30) and the mistake-prone nature of the application, I created this role to make it easier to manage the whole ordeal.

I made 2 design choices for this project:

* Keep the firmware as stock as possible, so not to introduce points of failure, source for lack of updates or vectors of attack. Not installing python to allow Ansible usage required to create the configuration files locally first, and **cat**'em into their actual file.

* Keeping as much of the firewall configuration in OpenWRT's web native language and window. This allows for less network-savvy coworkers to still check and modify the configuration if they need to. I managed to set everything in there, except for the forwarding between zones **on specified ports only**. That got to be put in the _firewall.user_ file.

Role Variables and defaults
---------------------------

* __OpenPorts__: '18001,18003,18010:18019'  
Ports to be allowed to cross over from client zone to backbone.
* __SSHPort__: 22
* __WebPort__: 80
* __MgmNetwork__: '10.1.10.0/24'  
Network that will be allowed to SSH into and manage the router via its web interface.
* __MonitorIP__: 10.1.100.98  
IP of a monitor system, that will be allowed to ping the router (even though it is not on the Management network)
* __ClientMTU__: 9000
* __timezone__: 'America/Argentina/Cordoba'
* __domain__: 'example.org'
* __timeserver__: 'time.{{domain}}'

Dependencies
------------

None


Gotchas and fixes
-----------------

VMWare likes to swap interfaces around. This leads to interfaces not always (ie: never) getting the same number upon reboots, ruining the zone firewalling.

In order to overcome this, the router VM's interfaces are to be created with a specific MAC address (ie: not auto but manual). On boot, the router runs a script to rename the interfaces according to the final number of the MAC address of each interface. It's messy, but hey, it works!

The MAC Addresses are 00:50:56:00:[Router Number]:[Interface Number].

Put the following code in a /sbin/IFRenamer.sh (_chmod_ it to executable)

```bash
#!/bin/sh
/etc/init.d/network stop
for i in `ip add show |  grep -o -E ": eth[0-9]" | grep -o -E "eth[0-9]"`;
do
	#echo "IF $i, MAC:"
	d=`ip address show $i  | grep -o -E "00:50:56:00:[0-9]+:[0-9]+" | grep -o -e "[0-9]$"`
	#echo $d
	ip link set $i  name eth"$d""$d"
done

for i in `ip add show |  grep -o -E ": eth[0-9]+" | grep -o -E "h[0-9]"|grep -o -E "[0-9]"`;
do
	ip link set eth$i$i  name eth$i
done

/etc/init.d/network start
```

and add it into _/etc/rc.local_:

```bash
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.
/sbin/IFRenamer.sh
exit 0
```

Also, important: The Dropbear config file does not support multiple interfaces listed in the 'Option Interface' stanza. Setting more than one interface to "Management" type hangs the ssh service.

Inventory example
-----------------

```yaml
router1 ansible_ssh_host=10.99.99.4 ansible_ssh_user=root ansible_ssh_pass="{{router1_pass}}"
router2 ansible_ssh_host=10.99.99.5 ansible_ssh_user=root ansible_ssh_pass="{{router2_pass}}"
router3 ansible_ssh_host=10.99.99.6 ansible_ssh_user=root ansible_ssh_pass="{{router3_pass}}"
router4 ansible_ssh_host=10.99.99.7 ansible_ssh_user=root ansible_ssh_pass="{{router4_pass}}"
router5 ansible_ssh_host=10.99.99.8 ansible_ssh_user=root ansible_ssh_pass="{{router5_pass}}"

[Routers]
router1
router2
router3
router4
router5
```

Playbook example
----------------

```yaml
---
- hosts: Routers
  gather_facts: no
  vars_files:
    - ../credentials/routers.yml
  roles:
    - ../roles/OpenWRTConfig
```

Router host_vars example
------------------------

Each router gets a file for its networks and interfaces. They are to be set up with this syntax.  
**WARNING: the network names do not support dashes ("-").**

```yaml
---
interfaces:
  backbone_net:
    interface: eth0
    ip: 172.20.0.101
    mask: 255.255.255.0
    type: backbone
  client_net_1:
    interface: eth1
    ip: 192.168.192.1
    mask: 255.255.252.0
    type: client
   client_net_2:
    interface: eth2
    ip: 192.168.206.1
    mask: 255.255.255.128
    type: client
   client_net_3:
    interface: eth3
    ip: 192.168.206.129
    mask: 255.255.255.192
    type: client
  Management:
    interface: eth4
    ip: '{{ansible_ssh_host}}'
    mask: 255.255.252.0
    type: management
    dns: 10.1.4.111
    gateway: 10.1.104.200
```

OpenWRT router virtual machine creation
---------------------------------------

* https://openwrt.org/docs/guide-user/virtualization/vmware
* https://www.gilesorr.com/blog/resize-mount-vdi.html
* https://www.gilesorr.com/blog/virtualbox-openwrt.html
* https://www.gilesorr.com/blog/vbox-openwrt-2018.html


License
-------

GNU GPLv3

Author Information
------------------

https://www.linkedin.com/in/franciscofregona
