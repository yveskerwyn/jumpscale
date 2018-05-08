# Zero-OS Primitives

## Test data

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

Or on OpenvCloud:
```
ipxe: https://bootstrap.gig.tech/ipxe/development/17d709436c5bc232/organization=zos-training-org%20development
```

=> https://ch-lug-dc01-001.gig.tech/CBGrid/Cloud%20Space?id=126
10.147.18.166


In case of not using a Zero-Tier en no IYO organization:
```
ipxe: https://bootstrap.gig.tech/ipxe/development/0/development

```

Used during tests:
```
ipxe: https://bootstrap.gig.tech/ipxe/development-update-e2fs/0/development
```

## Boot the Zero-OS node on OpenvCloud:

Get a cloud space on an OpenvCloud environment:
```python
ovc_url = 'ch-lug-dc01-001.gig.tech'
ovc_location = 'ch-gen-1'
ovc_instance_name = 'switserland'
ovc_cfg = dict(address=ovc_url, location=ovc_location)

#ovc_config_instance = j.tools.configmanager.configure(location="j.clients.openvcloud", instance=ovc_instance_name, data=ovc_cfg)

ovc_client = j.clients.openvcloud.get(instance=ovc_instance_name, data=ovc_cfg)
#ovc_client = j.clients.openvcloud.get(instance=ovc_instance_name)

ovc_account_name = 'Account_of_Yves'
vdc_name = 'zero-space'

ovc_account = ovc_client.account_get(name=ovc_account_name, create=False)
cloud_space = ovc_account.space_get(name=vdc_name, create=True)
#cloud_space = ovc_account.space_get(name=vdc_name, create=False)
```

Get the cloud space public IP address:
```python
cloudspace_public_ip_address = cloud_space.ipaddr_pub
```


Boot a VM with Zero-OS in the cloud space:
```python
vm_name = 'zero-os-yves'
iyo_organization = 'zos-training-org'
zos_kernel_params = ['organization={}'.format(iyo_organization), 'development']
#zos_branch = 'development'
zos_branch = 'development-update-e2fs'
zt_network_id = '0' # no ZeroTier

ipxe_url = 'ipxe: https://bootstrap.gig.tech/ipxe/{}/{}/'.format(zos_branch, zt_network_id) + '%20'.join(zos_kernel_params)

zos_vm = cloud_space.machine_create(name=vm_name, memsize=8, disksize=50, image='IPXE Boot', authorize_ssh=False, userdata=ipxe_url)
```

In case you don't use a ZeroTier, you need a port forward for the Redis (6379) and for the Open vSwitch container (9900)
```python
zos_vm.portforward_create(publicport=6379, localport=6379)
zos_vm.portforward_create(publicport=9900, localport=9900)
```

You also need to attach the external network directly to the VM, making it available for the Gateway container:
```python 
zos_vm.externalnetwork_attach()
```

We need the MAC address of the external network interface for configuring the Open vSwitch container later below:
```python
external_network_ip_address = zos_vm.model['interfaces'][1]['ipAddress']
external_network_mac_address = zos_vm.model['interfaces'][1]['macAddress']
```

Also get the IP address of the internet gateway:
```python
external_gw_ip_address = zos_vm.model['interfaces'][1]['params'].split()[0].rsplit(':')[1]
```

## Set up connection

Getting started:
```python
zos_instance_name = vm_name

# Existing connection
#j.clients.zero_os.list()
#zos_client = j.clients.zos.get(instance=zos_instance_name)

# New connection - first get your JWT
iyo_client = j.clients.itsyouonline.get(instance='main')
memberof_scope = 'user:memberof:{}'.format(iyo_organization)
jwt = iyo_client.jwt_get(scope=memberof_scope, refreshable=True)

# New connection - then connect
#node_address = '10.147.18.166'
node_address = cloudspace_public_ip_address

zos_cfg = {"host": node_address, "port": 6379, "password_": jwt}
zos_client = j.clients.zos.get(instance=zos_instance_name, data=zos_cfg)

# List the containers using the node interface
zos_node = j.clients.zos.sal.get_node(instance=zos_instance_name)
zos_node.containers.list()
```

To check the flist version that was used:
```python
zos_node.client.info.version()
```

Configure the backplane bridge - not relevant in case you have only one node??? but needed in order to deploy the beloz GW: 
```python
ovs_container_name = 'ovs'
zos_node.network.configure(cidr='192.168.69.0/24', vlan_tag=2312, ovs_container_name=ovs_container_name)
```

This creates another container, running Open vSwitch (`ovs`) bridging to the VLAN with tag `2312`:
```python
zos_node.containers.list()
ovs_container = zos_node.containers.get(name=ovs_container_name)
```

If used, setup the ZeroTier client:
```python
zt_instance_name = 'myzt'
zt_token = '**'
zt_cfg = dict([('token_', zt_token)])
zt_cl = j.clients.zerotier.get(instance=zt_instance_name, data=zt_cfg)
#zt_cl = j.clients.zerotier.get(instance=zt_instance_name)
```

## Gateway

Create a Gateway:
```python
gw_name = 'my-little-gw'
gw = zos_node.primitives.create_gateway(name=gw_name)
```

> At this point the actual container is not yet created, this only happens later when executing `gw_container.deploy()`

Define a network with name 'public' using a vlan:
```python
vlan_tag = 0
public_network_name = 'public'
public_net = gw.networks.add(name=public_network_name, type_='vlan', networkid=vlan_tag)
public_net.ip.cidr = external_network_ip_address
public_net.ip.gateway = external_gw_ip_address 
public_net.hwaddr = external_network_mac_address
```

Define a network with name 'private':
```python
private_network_name = 'private'
private_net = gw.networks.add(name=private_network_name, type_='vxlan', networkid=vlan_tag)
private_net.ip.cidr = '192.168.103.1/24'
private_net.hosts.nameservers = ['1.1.1.1']
```

Deploy the gateway to the Zero-OS node
```python
gw.deploy()
```

There is no list gateways, since there is no state kept about this in the node, that's the job of the remote 0-robot, you simply need to now the container - once deployed:
```python
zos_node.containers.list()
gw_container = zos_node.containers.get(name=gw_name)
```

Check the result:
```python
zos_node.client.bash('ip -d l').get()
```

Serialize the gateway configuration to JSON: 
```python
gw_json_string = gw.to_json()    
```

This allows you to define a Gateway sal from JSON:                              
```python
gw = zos_node.primitives.from_json(type_="gateway", json=gw_json_string)
```

Loop through the gateway networks:
```python
for net in gw.networks:
    print("Network %s uses subnet %s & netmask %s" % (net.name, net.ip.subnet, net.ip.netmask))
    print("Gateway ip in network %s is %s" % (net.name, net.ip.subnet))
```

In order to remove a network from the gateway:
```python
gw.networks.remove(public_network_name) # remove network using its name 
gw.networks.remove(private_net) # remove network using its object
```

In order to delete the gateway from the Zero-OS node:
```python
zos_node.primitives.drop_gateway(name=gw_name)
```

Or:
```python
gw_container.stop()
```

## Virtual Machine

First create an SSH key for authenticating against the VM:
```python
sshkey_name = "my_sshkey"
sshkey_path = "/root/.ssh/{}".format(sshkey_name)

sshkey_client = j.clients.sshkey.key_generate(path=sshkey_path)
```

> Make sure to take one of the gig-booteable flists: https://hub.gig.tech/gig-bootable

Create (define) a new Ubuntu virtual machine:
```python
vm_name = 'myvm'
#vm = zos_node.primitives.create_virtual_machine(name=vm_name, type_='ubuntu:latest')
#vm.flist = 'https://hub.gig.tech/gig-bootable/ubuntu:16.04.flist'
vm = zos_node.primitives.create_virtual_machine(name=vm_name, type_='ubuntu:lts')

#zos_node.primitives.drop_virtual_machine(name=vm_name)

#vm.memory = 1024 
#vm.cpu_nr = 1
```

Authorize the previously created key:
```python
vm.configs.add(name='mysshkey', path='/root/.ssh/authorized_keys', content=sshkey_client.pubkey)
```

Assign a IP address to the VM and create a user:
```python
host = private_net.hosts.add(host=vm, ipaddress='192.168.103.2')
host.cloudinit.users.add('gig', 'rooter')
```

Enable HTTP access and create portforwarding for HTTP and SSH:  
```python
gw.httpproxies.add(name='myproxy', host=public_net.ip.address, destinations=['http://192.168.103.2:8080'], types=['http'])
gw.portforwards.add(name='myforward1', source=(public_net.ip.address, 8080), target=('192.168.103.2', 8080))
gw.portforwards.add(name='myforward2', source=(public_net.ip.address, 7122), target=('192.168.103.2', 22))
```

In case you need to remove this:
```python
gw.httpproxies.remove('myproxy')
gw.portforwards.remove('myforward1')
gw.portforwards.remove('myforward2')
```

Update the gateway:
```python
gw.deploy()
```

Deploy the virtual machine:
```python
vm.deploy()
```


## Zero-DB

As a next step we will add a disk to the virtual machine. This requires us to first create a Zero-DB.

In order to list the physical disks on the Zero-OS node:
```python
zos_node.disks.list()
```

Create a Zero-DB:
```python
zdb_name = 'myzdb'
zdb_dir = '/var/cache/zdb'
zos_client.filesystem.mkdir(path=zdb_dir)
zdb = zos_node.primitives.create_zerodb(name=zdb_name, path=zdb_dir)
```

In order to delete the Zero-DB do one of the following:
```python
zos_node.primitives.drop_zerodb(name=zdb_name)
zdb_container = zos_node.containers.get(instance=zdb_name).stop()
```

Optionally create a namespace:
```python
namespace_name = 'my-namespace'
namespace = zdb.namespaces.add(name=namespace_name)
namespace.size = 20 # set namespace size
namespace.password = 'secret' # set namespace password
```

NOT NEEDED - is done automatically when deploying the ZDB
```python
nft = zos_node.client.nft
nft.open_port(9900)
```

Deploy the new namespace to the zdb
```python
zdb.deploy()
```

Check again the nft, 9900 will be added, next to 6379 (Redis) en 6600 (Node Robot):
```python
nft = zos_node.client.nft
nft.list()
```

Get namespace information:
```python
# List all namespaces 
zdb.namespaces.list()

# If no namespace was created explicitly, one will have been created with the default name of your disk - later:
namespace = zdb.namespaces['mydisk']
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
```

## Disk

Create (define) a disk:
```python
disk_name = 'mydisk'
disk = zos_node.primitives.create_disk(name=disk_name, zdb=zdb, mountpoint='/mountpointinsidevm', filesystem='btrfs') 
```

Deploy the disk, will create namespace on zdb
```python
disk.deploy()
```

You need to restart the VM at this point, or later?
```python
zos_vm.restart()
```

Attach the disk:
```python
zdisk.deploy()
vm.disks.add(name_or_disk=disk_name , url=zdisk)
vm.deploy()
```

## Misc

```python

# Start the vm
vm.start()

# Pause the vm
vm.pause()

# Stop the vm
vm.stop()

# Serialize vm to json
ubuntu_json_string = ubuntu_vm.to_json()

# Deserialize vm from json
ubuntu_vm = node.primitives.from_json(type="vm", json=ubuntu_json_string)

# Deleting the vm 
node.primitives.drop_vm("my-little-ubuntu-vm")


```


