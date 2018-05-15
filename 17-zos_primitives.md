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
- [Create and attach a virtual disk](#create-vdisk)
- [Test HTTP access](#http-access)
- [Create a private network using ZeroTier instead of VXLAN](#private-zt)


<a id="create-zt-network"></a>

## Create the ZeroTier network

Install ZT if needed:
```python
j.tools.prefab.local.network.zerotier.install()
j.tools.prefab.local.network.zerotier.start()
```

Set the name of the ZeroTier configuration instance for your ZeroTier account:
```python
zt_config_instance_name = 'my_zt_account'
```

In case you have already created a configuration instance for your ZeroTier account just get it:
```python
zt_client = j.clients.zerotier.get(instance=zt_config_instance_name)
```

Optionally, in order to delete your existing ZeroTier configuration instance:
```python
j.clients.zerotier.delete(instance=zt_config_instance_name)
```

In case you need to create a (new) JumpScale client for ZeroTier:
```python
zt_token = '***'
zt_cfg = dict([('token_', zt_token)])
zt_client = j.clients.zerotier.get(instance=zt_config_instance_name , data=zt_cfg)
```

In order to list all your available ZeroTier networks:
```python
zt_client.networks_list()
```

Set the name of the ZeroTier network you want to use as your management network:
Create the network:
```python
zt_admin_network_name = 'admin_network'
```

If this ZeroTier network was already created before, get it using the network id:
```python
zt_admin_network_id = '9f77fc393eb910bc'
zt_admin_network = zt_client.network_get(network_id=zt_admin_network_id)
```

If not created yet the network, create it:
```python
zt_admin_network = zt_client.network_create(public=False, name=zt_admin_network_name, auto_assign=True)
zt_admin_network_id = zt_admin_network.id
```

Since the current ZeroTier client doesn't allow you to set/update the auto-assign range you'll have to do it manually through https://my.zerotier.com, and check the result from JumpScale:
```python
zt_admin_network = zt_client.network_get(network_id=zt_admin_network_id)
zt_admin_network.config['ipAssignmentPools']
```

<a id="join-zt-network"></a>

## Join the ZeroTier network

Join:
```python
j.tools.prefab.local.network.zerotier.network_join(network_id=zt_admin_network_id)
```

Authorize the join request:
```python
zt_machine_addr = j.tools.prefab.local.network.zerotier.get_zerotier_machine_address()

zos_member = zt_admin_network.member_get(address=zt_machine_addr)
zos_member.authorize()
```

<a id="boot-zos"></a>

## Boot the Zero-OS node on OpenvCloud

List all your OpenvCloud configuration clients:
```python
j.clients.openvcloud.list()
```

Set the name of the OpenvCloud client to want to use:
```python
ovc_instance_name = 'swiss'
```

In case this client already exists, get it:
```python
ovc_client = j.clients.openvcloud.get(instance=ovc_instance_name)
```

Or create a new one:
```python
ovc_url = 'ch-lug-dc01-001.gig.tech'
ovc_location = 'ch-gen-1'
ovc_cfg = dict(address=ovc_url, location=ovc_location)

ovc_client = j.clients.openvcloud.get(instance=ovc_instance_name, data=ovc_cfg)
```

Get to your OpenvCloud account:
```python
ovc_account_name = 'Account_of_Yves'
ovc_account = ovc_client.account_get(name=ovc_account_name, create=False)
```

Create/get a cloud space (virtual datacenter)
```python
vdc_name = 'my-zero-space'
cloud_space = ovc_account.space_get(name=vdc_name, create=True)
```

Get the public IP address of the cloud space:
```python
cloudspace_public_ip_address = cloud_space.ipaddr_pub
```

Boot a VM with Zero-OS in the cloud space:
```python
vm_name = 'my-zos-vm'
iyo_organization = 'zos-training-org'
zos_kernel_params = ['organization={}'.format(iyo_organization), 'development']
zos_branch = 'development'

ipxe_url = 'ipxe: https://bootstrap.gig.tech/ipxe/{}/{}/'.format(zos_branch, zt_admin_network_id) + '%20'.join(zos_kernel_params)

zos_vm = cloud_space.machine_create(name=vm_name, memsize=8, disksize=10, datadisks=[50], image='IPXE Boot', authorize_ssh=False, userdata=ipxe_url)
```

If you specified a ZeroTier network, authorize the join request from the Zero-OS node:
```python
zos_member = zt_admin_network.member_get(public_ip=cloudspace_public_ip_address)
zos_member.authorize()
```

Attach an external network directly to the VM, we will need it for gateway later:
```python 
zos_vm.externalnetwork_attach()
```

We need the IP and MAC address of Zero-OS node on the OpenvCloud external network interface, for connecting to the gateway:
```python
external_network_ip_address = zos_vm.model['interfaces'][1]['ipAddress']
external_network_mac_address = zos_vm.model['interfaces'][1]['macAddress']
```

> The above might fail if executed immediately after attaching the external network, in that case just wait a while and try again. 

Also get the IP address of the Internet gateway:
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

If you already have a config instance, get the client:
```python
zos_client = j.clients.zos.get(instance=zos_instance_name, interactive=False)
```

If not, first get your JWT:
```python
iyo_client = j.clients.itsyouonline.get(instance='main')
memberof_scope = 'user:memberof:{}'.format(iyo_organization)
jwt = iyo_client.jwt_get(scope=memberof_scope, refreshable=True)
```

Create a new connection:
```python
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
sshkey_name = 'my_sshkey'
sshkey_path = "/root/.ssh/{}".format(sshkey_name)

sshkey_client = j.clients.sshkey.key_generate(path=sshkey_path)
#sshkey_client = j.clients.sshkey.get(sshkey_name)
```

 Authorize the new SSH key:
```python
pubkey = sshkey_client.pubkey
zos_node.client.bash('echo "{}" >> /root/.ssh/authorized_keys'.format(pubkey)).get()
```

Open port 22 for SSH Access:
```python
zos_node.client.nft.open_port(22)
```

> The above will only allow SSH access on port 22 via the ZeroTier admin network.

In order to test SSH access, make sure to the authorized SSH key is loaded by ssh-agent:
```
ssh-add ~/.ssh./my_sshkey
ssh <zos_member.private_ip>
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

There is no API to list the gateways, since there is no state kept about this in the node, that's the job of the remote 0-robot, you simply need to now the container - once deployed:
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

This allows you to define a Gateway SAL from JSON:                              
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
vm_name = 'vm'
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
gw.httpproxies.add(name='my_proxy', host=public_net.ip.address, destinations=['http://192.168.103.2:8080'], types=['http'])
```

<a id="port-forwards"></a>

## Add port forwards to virtual machine

In order to make the VM accessible from the external network on HTTP en SSH, add following port forwards:
```python
gw.portforwards.add(name='my_forward1', source=(public_net.ip.address, 8080), target=('192.168.103.2', 8080))
gw.portforwards.add(name='my_forward2', source=(public_net.ip.address, 7122), target=('192.168.103.2', 22))
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
zdb_name = 'zdb'
zdb_dir = '/var/cache/zdb'
zos_client.filesystem.mkdir(path=zdb_dir)
zdb = zos_node.primitives.create_zerodb(name=zdb_name, path=zdb_dir)
```

In order to delete the Zero-DB do one of the following:
```python
zos_node.primitives.drop_zerodb(name=zdb_name)
zdb_container = zos_node.containers.get(name=zdb_name).stop()
```

NOT NEEDED - is done automatically when deploying the your first virtual isk:
```python
nft = zos_node.client.nft
nft.open_port(9900)
```

Deploy the new namespace to the zdb
```python
zdb.deploy()
```

<a id="create-vdisk"></a>

## Create and attach a virtual disk

Create (define) a disk:
```python
vdisk_name = 'vdisk'
vdisk = zos_node.primitives.create_disk(name=vdisk_name, zdb=zdb, mountpoint='/mnt', filesystem='btrfs') 
#zdisk.mountpoint = '/mnt'
```

Deploy the disk, will create namespace on zdb
```python
vdisk.deploy()
```

In case you already deployed the virtual machine, you will need to shutdown it before attaching the disk:
```python
vm.shutdown()
```

Attach the disk:
```python
vm.disks.add(name_or_disk=vdisk)
vm.deploy()
```

check the result:
```python
zos_node.client.bash('virsh list').get()
```

<a id="http-access"></a>

## Test HTTP access

Make sure the previously created SSH key is loaded by the SSH agent:
```bash
ssh-add ~/.ssh/my_sshkey 
```

SSH into the virtual machine:
```bash
ssh <external_network_ip_address> -p7122
```

Start a HTTP server on the virtual machine:
```bash
python3 -m http.server 8080
```

<a id="private-zt"></a>
## Create a private network using ZeroTier instead of VXLAN

Create/get a new ZeroTier network:
```python
zt_app_network_name = 'app_network'

zt_app_network = zt_client.network_create(public=False, name=zt_app_network_name , auto_assign=True)
zt_app_network_id = zt_app_network.id
```

Manually select a auto-assign range.

Drop the existing GW:
```python
zos_node.primitives.drop_gateway(name=gw_name)
```

Create a new one:
```python
gw2_name = 'my-gw2'
gw2 = zos_node.primitives.create_gateway(name=gw2_name)
```

Define the public leg, same as before:
```python
vlan_tag = 0
public_network_name = 'public'
public_net = gw2.networks.add(name=public_network_name, type_='vlan', networkid=vlan_tag)
public_net.ip.cidr = external_network_ip_address
public_net.ip.gateway = external_gw_ip_address 
public_net.hwaddr = external_network_mac_address
```

Define the ZeroTier private network:
```python
private_network_name = 'zt'
private_net = gw2.networks.add(name=zt_app_network_name, type_='zerotier', networkid=zt_app_network_id)
private_net.hosts.nameservers = ['1.1.1.1']
```

Define a new VM:
```python
vm2_name = 'vm2'
vm2 = zos_node.primitives.create_virtual_machine(name=vm2_name, type_='ubuntu:lts')

vm2.configs.add(name='mysshkey', path='/root/.ssh/authorized_keys', content=sshkey_client.pubkey)
```

Deploy a new disk:
```python
vdisk2_name = 'vdisk2'
vdisk2 = zos_node.primitives.create_disk(name=vdisk2_name, zdb=zdb, mountpoint='/mnt', filesystem='btrfs')
vdisk2.deploy()
```

Deploy:
```python
gw2.deploy()
```

At this point the gateway will have send a join request for the specified ZeroTier network, authorize it:
```python
zt_app_network_address = zt_app_network.members_list()[0].address
zos_member = zt_app_network.member_get(address=zt_app_network_address)
zos_member.authorize()
```

Deploy the VM:
```python
vm2.deploy()
```

Shutdown:
```python
vm2.shutdown()
```

Add the machine to the private network:
```python
vm2.nics.add_zerotier(network=zt_app_network)
```

Deploy the VM again
```python
vm2.deploy()
```

The VM will automatically get authorized into the ZeroTier application network.

Get the private IP address of the newly added VM:
```python
zos_member = zt_app_network.member_get(name=vm2_name)
zt_addr2 = zos_member.private_ip
```

Configure port forwarding:
```python
gw2.portforwards.add(name='my_forward1', source=(public_net.ip.address, 8080), target=(zt_addr2, 8080))
gw2.portforwards.add(name='my_forward2', source=(public_net.ip.address, 7122), target=(zt_addr2, 22))
```

Redeploy the gateway:
```python
gw2.deploy()
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