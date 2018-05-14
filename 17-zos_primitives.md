# Getting started with the Zero-OS Primitives

- [Create the ZeroTier network](#create-zt-network)
- [Join the ZeroTier network](#join-zt-network)
- [Boot the Zero-OS node on OpenvCloud](#boot-zos)
- [Connect using the JumpScale client for Zero-OS](#zos-client)
- [Authorize SSH key in Zero-OS node](#authorize-sshkey)
- [Create an Open vSwitch (OVS) container](#ovs-container)
- [Create the Zero-OS gateway](#create-gw)
- [Create a virtual machine](#create-vm)
- [Authorize SSH key in virtual machine](#authorize-sshkey2)
- [Assign an IP address to the virtual machine](#assign-ip)
- [Configure reverse proxy](#reverse-proxy)
- [Add port forwards to virtual machine](#port-forwards)
- [Create Zero-DB](#create-zdb)
- [Create and attach a disk](#create-disk)
- [Test HTTP access](#http-access)


<a id="create-zt-network"></a>

## Create the ZeroTier network

```python
zt_network_id = '0' # if no ZeroTier
zt_network_id = '17d709436c5bc232'
local_prefab = j.tools.prefab.local
```

Install ZT if needed:
```python
local_prefab.network.zerotier.install()
local_prefab.network.zerotier.start()
```

Get a JumpScale client for ZeroTier:
```python
zt_instance_name = 'myzt'
zt_token = '***'
zt_cfg = dict([('token_', zt_token)])
zt_client = j.clients.zerotier.get(instance=zt_instance_name, data=zt_cfg)
#zt_client = j.clients.zerotier.get(instance=zt_instance_name)
```

Create the network:
```python
zt_network_name = 'my_zt_network'
zt_routes = [{'target': '10.147.18.0/24', 'via': None}]

zt_network = zt_client.network_create(public=False, name=zt_network_name, routes=zt_routes)
zt_network_id = zt_network.id

# zt_network.config['ipAssignmentPools']
```

<a id="join-zt-network"></a>

## Join the ZeroTier network

Join:
```python
j.tools.prefab.local.network.zerotier.network_join(network_id=zt_network_id)
```

Authorize the join request:
```python
zt_machine_addr = j.tools.prefab.local.network.zerotier.get_zerotier_machine_address()

#zt_network = zt_client.network_get(network_id=zt_network_id)

zos_member = zt_network.member_get(address=zt_machine_addr)
zos_member.authorize()
```

<a id="boot-zos"></a>

## Boot the Zero-OS node on OpenvCloud

Get a client for OpenvCloud:
```python
ovc_url = 'ch-lug-dc01-001.gig.tech'
ovc_location = 'ch-gen-1'
ovc_instance_name = 'switserland'
ovc_cfg = dict(address=ovc_url, location=ovc_location)

ovc_client = j.clients.openvcloud.get(instance=ovc_instance_name, data=ovc_cfg)
#ovc_client = j.clients.openvcloud.get(instance=ovc_instance_name)
```

Get to your OpenvCloud account:
```python
ovc_account_name = 'Account_of_Yves'
ovc_account = ovc_client.account_get(name=ovc_account_name, create=False)
```
Create/get a cloud space (virtual datacenter)
```python
vdc_name = 'zero-space'
cloud_space = ovc_account.space_get(name=vdc_name, create=True)
```

Get the public IP address of the cloud space:
```python
cloudspace_public_ip_address = cloud_space.ipaddr_pub
```

Boot a VM with Zero-OS in the cloud space:
```python
vm_name = 'zero-os-yves'
iyo_organization = 'zos-training-org'
zos_kernel_params = ['organization={}'.format(iyo_organization), 'development']
zos_branch = 'development'

ipxe_url = 'ipxe: https://bootstrap.gig.tech/ipxe/{}/{}/'.format(zos_branch, zt_network_id) + '%20'.join(zos_kernel_params)

zos_vm = cloud_space.machine_create(name=vm_name, memsize=8, disksize=10, datadisks=[50], image='IPXE Boot', authorize_ssh=False, userdata=ipxe_url)
```

If you specified a ZeroTier network, authorize the join request from the Zero-OS node:
```python
zos_member = zt_network.member_get(public_ip=cloudspace_public_ip_address)
zos_member.authorize()
```

In case you don't use a ZeroTier (`zt_network_id = 0`), you need a port forward for the Redis (6379) and optionaly for the Open vSwitch container (9900)
```python
zos_vm.portforward_create(publicport=6379, localport=6379)
#zos_vm.portforward_create(publicport=9900, localport=9900)
```

Attach an external network directly to the VM, making it available for the gateway container:
```python 
zos_vm.externalnetwork_attach()
```

We need the IP and MAC address of Zero-OS node on the external network interface, for connecting to the gataway:
```python
external_network_ip_address = zos_vm.model['interfaces'][1]['ipAddress']
external_network_mac_address = zos_vm.model['interfaces'][1]['macAddress']
```

Also get the IP address of the internet gateway:
```python
external_gw_ip_address = zos_vm.model['interfaces'][1]['params'].split()[0].rsplit(':')[1]
```

<a id="zos-client"></a>

## Connect using the JumpScale client for Zero-OS

Check the existing configuration instances:
```python
j.clients.zos.list()
```

Name of the Zero-OS configuration instance:
```python
zos_instance_name = vm_name
```

For an existing connection:
```python
zos_client = j.clients.zos.get(instance=zos_instance_name)
```

For a new connection, first get your JWT:
```python
iyo_client = j.clients.itsyouonline.get(instance='main')
memberof_scope = 'user:memberof:{}'.format(iyo_organization)
jwt = iyo_client.jwt_get(scope=memberof_scope, refreshable=True)
```

Create a new connection:
```python
if zt_network_id == 0:
    node_address = cloudspace_public_ip_address
else:
    node_address = zos_member.private_ip

zos_cfg = {"host": node_address, "port": 6379, "password_": jwt}
zos_client = j.clients.zos.get(instance=zos_instance_name, data=zos_cfg)
```
Get the node interface and list the containers:
```python
zos_node = j.clients.zos.sal.get_node(instance=zos_instance_name)
zos_node.containers.list()
```

To check the flist version that was used:
```python
zos_node.client.info.version()
```

<a id="authorize-sshkey"></a>

## Authorize SSH key in Zero-OS node

Since the machine was started development mode, you can SSH it.

First create a new SSH key, using the `sshkey` client:
```python
sshkey_name = "my_sshkey"
sshkey_path = "/root/.ssh/{}".format(sshkey_name)

sshkey_client = j.clients.sshkey.key_generate(path=sshkey_path)
#sshkey_client = j.clients.sshkey.get(sshkey_name)
```

 Authorize the new SSH key:
```python
#zos_node.client.bash('wget ssh.maxux.net/yveskerwyn -O - | ash').get()
zos_node.client.bash('echo "{}" >> /root/.ssh/authorized_keys'.format(key)).get()
zos_node.client.nft.open_port(22)
```

<a id="ovs-container"></a>

## Create an Open vSwitch (OVS) container

The Open vSwitch (OVS) container, needed in order to deploy the below GW: 
```python
ovs_container_name = 'ovs'
zos_node.network.configure(cidr='192.168.69.0/24', vlan_tag=2312, ovs_container_name=ovs_container_name)
```

This creates another container, running Open vSwitch (`ovs`) bridging to the VLAN with tag `2312`:
```python
zos_node.containers.list()
ovs_container = zos_node.containers.get(name=ovs_container_name)
```

<a id="create-gw"></a>

## Create the Zero-OS gateway

Create a Gateway:
```python
gw_name = 'my-gw'
gw = zos_node.primitives.create_gateway(name=gw_name)
```

> At this point the actual container is not yet created, this only happens later when executing `gw_container.deploy()`

Define a network with name `public` using a VLAN:
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
vlxan_tag = 100
private_net = gw.networks.add(name=private_network_name, type_='vxlan', networkid=vlxan_tag)
private_net.ip.cidr = '192.168.103.1/24'
private_net.hosts.nameservers = ['1.1.1.1']
```

Check:
```python
gw.to_dict()
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

<a id="create-vm"></a>

## Create a virtual machine

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

<a id="authorize-sshkey2"></a>

## Authorize SSH key in virtual machine

Authorize the previously created key:
```python
vm.configs.add(name='mysshkey', path='/root/.ssh/authorized_keys', content=sshkey_client.pubkey)
```

<a id="assign-ip"></a>

## Assign an IP address to the virtual machine

```python
host = private_net.hosts.add(host=vm, ipaddress='192.168.103.2')
#host.cloudinit.users.add('gig', 'rooter')
```

<a id="reverse-proxy"></a>

## Configure reverse proxy

```python
gw.httpproxies.add(name='myproxy', host=public_net.ip.address, destinations=['http://192.168.103.2:8080'], types=['http'])
```

<a id="port-forwards"></a>

## Add port forwards to virtual machine

```python
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

We will deploy the virtual machine later, after having added a disk:
```python
#vm.deploy()
```

<a id="create-zdb"></a>

## Create Zero-DB

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

<a id="create-disk"></a>

## Create and attach a disk

Create (define) a disk:
```python
zdisk_name = 'mydisk'
zdisk = zos_node.primitives.create_disk(name=zdisk_name, zdb=zdb, mountpoint='/mnt', filesystem='btrfs') 
#zdisk.mountpoint = '/mnt'
```

Deploy the disk, will create namespace on zdb
```python
zdisk.deploy()
```

In case you already deployed the virtual machine, you will need to shutdown it before attaching the disk:
```python
vm.shutdown()
```

Attach the disk:
```python
vm.disks.add(name_or_disk=zdisk)
vm.deploy()
```

<a id="http-access"></a>

## Test HTTP access

Make sure the previously created SSH key is loaded by the SSH agent:
```bash
ssh-add ~/.ssh/my_sshkey 
```

SSH into the virtual machine:
```bash
ssh <> -p7122
```

Start a HTTP server on the virtual machine:
```bash
python3 -m http.server 8080
```

## Misc

Optionally create a namespace:
```python
namespace_name = 'my-namespace'
namespace = zdb.namespaces.add(name=namespace_name)
namespace.size = 20 # set namespace size
namespace.password = 'secret' # set namespace password
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

```python
# Start the vm
vm.start()

# Pause the vm
vm.pause()

# Resume the vm
vm.resume()

# Stop the vm
vm.stop()

# Serialize vm to json
ubuntu_json_string = ubuntu_vm.to_json()

# Deserialize vm from json
ubuntu_vm = node.primitives.from_json(type="vm", json=ubuntu_json_string)

# Deleting the vm 
node.primitives.drop_vm("my-little-ubuntu-vm")
```