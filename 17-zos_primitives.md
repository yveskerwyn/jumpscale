# Zero-OS Primitives

Specifications:
https://github.com/Jumpscale/lib9/blob/zosprimitives/specs/zero-os-primitives.md

Tested from:
- https://be-gen-1.demo.greenitglobe.com/CBGrid/Virtual%20Machine?id=2841
- `ssh root@195.134.212.38 -p7722`

ZeroTier network:
https://my.zerotier.com/network/17d709436c5bc232

ItsYou.online organization:
https://itsyou.online/#/organization/zos-training-org/people

Bootstrap iPXE URL:
- https://bootstrap.gig.tech/ipxe/development/17d709436c5bc232/organization=zos-training-org%20development
- http://unsecure.bootstrap.gig.tech/ipxe/development/17d709436c5bc232/organization=zos-training-org%20development

Packet.net:
- name: zos-training-node:
- URL: https://app.packet.net/devices/3fe15ec2-eec5-4784-b2ed-e5a0cd3755f5


Pseudo code:
```python
node = j.clients.zeroos.sal.node_get('myzos')

# Create a new gateway
# Name will be used for creating the effective zero-os container
gw = node.primitives.create_gateway(name="my-little-gw")

# Add a network with name 'public' using a vlan [vlan/vxlan/zerotier]
pubnet = gw.networks.add(name="public", type_="vlan")
pubnet.type = 'vlan'                  # NEEDS UPDATE
pubnet.vlantag = 100                    # vlan to be used for the pubnet 
pubnet.ip.cidr = '185.69.166.2/24'      # ip/subnet to be used for the nic configuration 
                                        # of the gateway itself for pubnet
pubnet.ip.gateway = '185.69.166.254'    # gateway to be used for the nic configuration 
                                        # of the gateway itself for pubnet

# Add a network with name 'private' using a zerotier [vlan/vxlan/zerotier]
privnet = gw.networks.add(name="private", type_="zerotier")
privnet.networkid = 'jkhg53df3g'                  # If empty, zerotier network will be created 
                                                  # automatically using token
privnet.client = j.clients.zerotier.get("geert")  # Needed to create network and / or authorize 
                                                  # gateway & vms into network
privnet.ip.cidr = '192.168.0.1/24'                # ip/subnet to be used for the nic confguration
                                                  # of the gateway itself for privnet
privnet.hosts.nameservers = ['8.8.8.8']           # nameservers that will be used by the hosts in privnet
                                                  # if not set, defaults to ['8.8.8.8']

# Deploy the gateway to the zero-os node
gw.deploy()

# Serialize the gateway configuration to json
gw_json_string = gw.to_json()    # only contains gateway information, no vm details apart from host info
                                 # (see further down)

# Instantiate gateway sal from json
gw = node.primitives.from_json(type_="gateway", json=gw_json_string)

# Reference network with name public
pubnet = gw.networks['public']

# Loop through gateway networks
for net in gw.networks:
    print("Network %s uses subnet %s & netmask %s" % (net.name, net.ip.subnet, net.ip.netmask))
    print("Gateway ip in network %s is %s" % (net.name, net.ip.subnet))
    
# Remove a network from the gateway
gw.networks.remove("private") # remove network using its name 
gw.networks.remove(privnet) # remove network using its object

# Delete the gateway on the zero-os node
node.primitives.drop_gateway(name="my-little-gw")

# Create a new ubuntu vm [ubuntu:16.04/ubuntu:18.04/zero-os:1.2.1]
ubuntu_vm = node.primitives.create_virtual_machine(name="my-little-ubuntu-vm", type_='ubuntu:16.04')
ubuntu_vm.memory = 1024   # 1024 MiB
ubuntu_vm.cpu_nr = 1      # 1 vcpu

# Add disk named mark of type db [db, archive, temp]
db_disk = ubuntu_vm.disks.add(name="mark", type=db)
db_disk.size = 50   # 50 GiB
db_disk.fs = 'ext4'              # optional. possible types: [ext4, btrfs, xfs]
db_disk.mountpoint = '/mnt/mark'  # optional. must be used in combination of fs property
                                 # if both are set the disk will be formatted and mountpoint will be added # in /etc/fstab. hot mounting is not supported while the vm is running, 
                                 # hot adding the disk to the running vm is supported
db_disk.url = zdb.namespaces["my-namespace"]     # See below how the zdb object gets created, convenience option
db_disk.url = zdb.namespaces["my-namespace"].url # Can also set url directly
print(db_disk.zdb_namespace)                     # ==> always return url, for loosely coupling different sals

# Add vm to a network
privnet = gw.networks['private']
host = privnet.hosts.add_vm(host=ubuntu_vm, ipaddress='192.168.0.2')   # hot adding nics to a running vm 
                                                                    # is supported

host = privnet.hosts.add(name='myhostname', macaddress='54:42:01:02:03:04', ipaddress='192.168.0.2')   # add mac manually
host = privnet.hosts.add(name='my2ndhost', macaddress=privnet.get_free_mac(), ipaddress='192.168.0.2')   # generate mac for network
host.cloudinit.users.add(user='myuser', password='mypassword', sudo=False) # optional cloud init configuration
host.cloudinit.metadata['local-hostname'] = 'myhostname'

# cloudinit have userdata and metadata dict for custom cloud-init data
print(host.name)    # name of vm object

# privnet.hosts also supports indexing on host name (ubuntuvm.name), listing, and removing hosts

# Redeploy the gw to add the host for real
gw.deploy()

# Deploy the vm
ubuntu_vm.deploy()

# Start the vm
ubuntu_vm.start()

# Pause the vm
ubuntu_vm.pause()

# Stop the vm
ubuntu_vm.stop()

# Serialize vm to json
ubuntu_json_string = ubuntu_vm.to_json()

# Deserialize vm from json
ubuntu_vm = node.primitives.from_json(type="vm", json=ubuntu_json_string)

# Deleting the vm 
node.primitives.drop_vm("my-little-ubuntu-vm")

# Create a new zero-os vm
zeroos_vm = node.primitives.create_virtual_machine(name="my-little-ubuntu-vm", type_='zero-os')
zeroos_vm.ipxe_url = 'https://bootstrap.gig.tech/ipxe/master/abcef01234567890/organization=myorg'
zeroos_vm.memory = 1024   # 1024 MiB
zeroos_vm.cpu_nr = 1      # 1 vcpu

# Add disk named mark of type db [db, archive, temp]
db_disk = zeroos_vm.disks.add(name="mark", url='zdb://172.18.0.1:9900?size=10G&blocksize=4096&namespace=mydisk')

# Add a portforward from the public network to the private network to host 
pubnet = gw.networks['public']
privnet = gw.networks['private']
pfwd = gw.portforwards.add(name='http')
pfwd.source.ipaddress = '173.17.22.22'
pfwd.source.port = 80
pfwd.target.ipaddress = '192.168.0.2'
pfwd.target.port = 8080
gw.portforwards.add(name='https', ('173.17.22.22': 443), ('192.168.0.3', 443))
gw.deploy() # deploy to make changes have effect
# ... gw.portforwards also alows indexing on portforward name, listing and removing portforwards

# ... add similar functions for reverse proxy configuration

# Create a new zdb
# Name will be used for creating the effective zero-os container
zdb = node.primitives.create_zdb(name="my-zdb", disk='/mnt/zerodbs/vda')

# create namespace
namespace = zdb.namespaces.add('my-namespace')
namespace.size = 20 # set namespace size
namespace.password = 'secret' # set namespace password

# deploy the new namespace to the zdb
zdb.deploy()

# get namespace information
info = namespace.info()

# get namespace url for kvm
url = namespace.url

# loop through the namespaces
for namespace in zdb.namespaces:
    print(namespace.size)

# get a namespace by name
namespace = zdb.namespaces["my-namespace"]
print(namespace.size)

# delete namespace
zdb.namespaces.remove("my-namespace")  # Delete a namespace using its name
zdb.namespaces.remove(namespace)       # Delete a namespace using object reference

# Deleting a complete zbd
node.primitives.drop_zdb("my-zdb")

# Alternative way of adding a disk
zdisk = node.primitives.create_disk('mydisk', zdb, '/mountpointinsidevm', 'ext4', 20)
zddisk.deploy() # will create namespace on zdb and deploy it
ubuntu_vm.disks.add('mydisk', zdisk)
ubuntu_vm.deploy()
```

Getting started:
```python
zos_instance_name = 'zos-training-node'

# Existing connection
#j.clients.zero_os.list()
zos_client = j.clients.zero_os.get(instance=zos_instance_name)

# New connection - first get your JWT
iyo_organization = "zos-training-org"
iyo_client = j.clients.itsyouonline.get(instance='main')
memberof_scope = "user:memberof:{}".format(iyo_organization)
jwt = iyo_client.jwt_get(scope=memberof_scope, refreshable=True)

# New connection - then connect
node_address = '10.147.18.213'
zos_cfg = {"host": node_address, "port": 6379, "password_": jwt}
zos_client = j.clients.zero_os.get(instance=zos_instance_name, data=zos_cfg)

# List the containers using the node interface
zos_node = j.clients.zero_os.sal.get_node(instance=zos_instance_name)
zos_node.containers.list()
```

Setup the ZeroTier client:
```python
zt_instance_name = 'my_zt'
zt_token = 'tFGYkdKutMR3crYA9FHfzVKKhMdbanDq'
zt_cfg = dict([("token_", zt_token)])
zt_cl = j.clients.zerotier.get(instance=zt_instance_name, data=zt_cfg)
#zt_cl = j.clients.zerotier.get(instance=zt_instance_name)
```


Gateway:
```python
gw_name = 'my-little-gw'
gw = zos_node.primitives.create_gateway(name=gw_name)

# There is no list gateways, since there is not state kept about this in the node, that's the job of the remote 0-robot, you simply need to now the container:
gw_container = zos_node.containers.get(name=gw_name)
#zos_node.primitives.drop_gateway(name=gw_name)
#OR: gw_container.stop()

# Add a network with name 'public' using a vlan
vlan_tag = 101
public_network_name = 'public'
pubnet = gw.networks.add(name=public_network_name, type_='vlan', networkid=vlan_tag)

#pubnet.type = 'public'                  # one of [public/private], is used for firewall configuration
#pubnet.vlantag = 100                    # vlan to be used for the pubnet 
pubnet.ip.cidr = '185.69.166.2/24'      # ip/subnet to be used for the nic configuration of the gateway itself for pubnet
pubnet.ip.gateway = '185.69.166.254'    # gateway to be used for the nic configuration 
                                        # of the gateway itself for pubnet

# Add a network with name 'private' using a zerotier [vlan/vxlan/zerotier]
private_zt_id = "9f77fc393e24babd"
private_network_name = "private"
privnet = gw.networks.add(name=private_network_name, type_="zerotier", networkid=private_zt_id)
#privnet.networkid = 'jkhg53df3g'                 # If empty, zerotier network will be created 
                                                  # automatically using token
privnet.client = j.clients.zerotier.get(instance=zt_instance_name)  # Needed to create network and / or authorize 
                                                  # gateway & vms into network
privnet.ip.cidr = '192.168.0.1/24'                # ip/subnet to be used for the nic configuration of the gateway itself for privnet
privnet.hosts.nameservers = ['8.8.8.8']           # nameservers that will be used by the hosts in privnet
                                                  # if not set, defaults to ['8.8.8.8']

# had to add this, in order to create an Open vSwitch container: 
zos_node.network.configure(cidr='192.168.69.0/24', vlan_tag=2312, ovs_container_name='ovs')


# Deploy the gateway to the zero-os node
gw.deploy()
```

In order to get the preserve the 
```python
# Serialize the gateway configuration to json
gw_json_string = gw.to_json()    # only contains gateway information, no vm details apart from host info
                                 # (see further down)                               


# Instantiate gateway sal from json
gw = zos_node.primitives.from_json(type_="gateway", json=gw_json_string)
```


```python
# Reference network with name public
pubnet = gw.networks['public']

# Loop through gateway networks
for net in gw.networks:
    print("Network %s uses subnet %s & netmask %s" % (net.name, net.ip.subnet, net.ip.netmask))
    print("Gateway ip in network %s is %s" % (net.name, net.ip.subnet))
    
# Remove a network from the gateway
gw.networks.remove("private") # remove network using its name 
gw.networks.remove(privnet) # remove network using its object

# Delete the gateway on the zero-os node
zos_node.primitives.drop_gateway(name=gw_name)
```

Create a new Ubuntu vm:
```python
# [ubuntu:16.04/ubuntu:18.04/zero-os:1.2.1]
vm_name = "my-little-ubuntu-vm"
ubuntu_vm = zos_node.primitives.create_virtual_machine(name=vm_name, type_='ubuntu:16.04')
ubuntu_vm.memory = 1024   # 1024 MiB
ubuntu_vm.cpu_nr = 1      # 1 vcpu
```

Create a Zero-DB:
```python
zdb_name = 'myzdb'
zdb_dir = '/var/cache/zdb'
#zos_node.disks.list()
zos_client.filesystem.mkdir(path=zdb_dir)
zdb = zos_node.primitives.create_zerodb(name=zdb_name, path=zdb_dir)
#zos_node.primitives.drop_zerodb(name=zdb_name)
#OR: zdb_container=zos_node.containers.get(instance=zdb_name).stop()
```

Create name space:
```python
namespace_name = 'my-namespace'
namespace = zdb.namespaces.add(name=namespace_name )
namespace.size = 20 # set namespace size
namespace.password = 'secret' # set namespace password

# NOT NEEDED - is done automatically
nft = zos_node.client.nft
nft.open_port(9900)

# deploy the new namespace to the zdb
zdb.deploy()

# get namespace information
info = namespace.info()

# get namespace url for kvm
url = namespace.url

# loop through the namespaces
for namespace in zdb.namespaces:
    print(namespace.size)

# get a namespace by name
namespace = zdb.namespaces[namespace_name]
print(namespace.size)

# delete namespace
zdb.namespaces.remove(item=namespace_name)  # Delete a namespace using its name
zdb.namespaces.remove(item=namespace)       # Delete a namespace using object reference

# Deleting a complete zbd
node.primitives.drop_zdb("my-zdb")
```

Create a disk:
```python
disk_name = 'mydisk'
zdisk = zos_node.primitives.create_disk(name=disk_name, zdb=zdb, mountpoint='/mountpointinsidevm', filesystem='btrfs', size=20)
zdisk.deploy() # will create namespace on zdb and deploy it
ubuntu_vm.disks.add(name=disk_name , url_or_disk=zdisk)

ubuntu_vm.deploy()
```





```python
# Add disk named mark of type db [db, archive, temp]
db_disk = ubuntu_vm.disks.add(name="mark", url_or_disk="??")
db_disk.size = 50   # 50 GiB
db_disk.fs = 'ext4'              # optional. possible types: [ext4, btrfs, xfs]
db_disk.mountpoint = '/mnt/mark'  # optional. must be used in combination of fs property
                                 # if both are set the disk will be formatted and mountpoint will be added # in /etc/fstab. hot mounting is not supported while the vm is running, 
                                 # hot adding the disk to the running vm is supported
db_disk.url = zdb.namespaces["my-namespace"]     # See below how the zdb object gets created, convenience option
db_disk.url = zdb.namespaces["my-namespace"].url # Can also set url directly
print(db_disk.zdb_namespace)                     # ==> always return url, for loosely coupling different sals

# Add vm to a network
privnet = gw.networks['private']
host = privnet.hosts.add_vm(host=ubuntu_vm, ipaddress='192.168.0.2')   # hot adding nics to a running vm 
                                                                    # is supported

host = privnet.hosts.add(name='myhostname', macaddress='54:42:01:02:03:04', ipaddress='192.168.0.2')   # add mac manually
host = privnet.hosts.add(name='my2ndhost', macaddress=privnet.get_free_mac(), ipaddress='192.168.0.2')   # generate mac for network
host.cloudinit.users.add(user='myuser', password='mypassword', sudo=False) # optional cloud init configuration
host.cloudinit.metadata['local-hostname'] = 'myhostname'

# cloudinit have userdata and metadata dict for custom cloud-init data
print(host.name)    # name of vm  object

# privnet.hosts also supports indexing on host name (ubuntuvm.name), listing, and removing hosts

# Redeploy the gw to add the host for real
gw.deploy()

# Deploy the vm
ubuntu_vm.deploy()

# Start the vm
ubuntu_vm.start()

# Pause the vm
ubuntu_vm.pause()

# Stop the vm
ubuntu_vm.stop()

# Serialize vm to json
ubuntu_json_string = ubuntu_vm.to_json()

# Deserialize vm from json
ubuntu_vm = node.primitives.from_json(type="vm", json=ubuntu_json_string)

# Deleting the vm 
node.primitives.drop_vm("my-little-ubuntu-vm")

# Create a new zero-os vm
zeroos_vm = node.primitives.create_virtual_machine(name="my-little-ubuntu-vm", type_='zero-os')
zeroos_vm.ipxe_url = 'https://bootstrap.gig.tech/ipxe/master/abcef01234567890/organization=myorg'
zeroos_vm.memory = 1024   # 1024 MiB
zeroos_vm.cpu_nr = 1      # 1 vcpu

# Add disk named mark of type db [db, archive, temp]
db_disk = zeroos_vm.disks.add(name="mark", url='zdb://172.18.0.1:9900?size=10G&blocksize=4096&namespace=mydisk')

# Add a portforward from the public network to the private network to host 
pubnet = gw.networks['public']
privnet = gw.networks['private']
pfwd = gw.portforwards.add(name='http')
pfwd.source.ipaddress = '173.17.22.22'
pfwd.source.port = 80
pfwd.target.ipaddress = '192.168.0.2'
pfwd.target.port = 8080
gw.portforwards.add(name='https', ('173.17.22.22': 443), ('192.168.0.3', 443))
gw.deploy() # deploy to make changes have effect
# ... gw.portforwards also alows indexing on portforward name, listing and removing portforwards

# ... add similar functions for reverse proxy configuration
```


