
# Getting started with JumpScale 9.3.0 Demo Script

- [Installation of JumpSale](#installation)
- [SAL](sal)
- [Prefab](#prefab)
- [Config Manager](#config-manager)
- [Clients](#clients)
- [Node Manager](#nodemanager)
- [Executor](#executor)

## Installation of JumpSale

From command line:
```bash
sudo -i
curl https://raw.githubusercontent.com/Jumpscale/bash/development/install.sh?$RANDOM > /tmp/install.sh;bash /tmp/install.sh
source ~/.bashrc
export JS9BRANCH=development
ZInstall_host_js9_full
```

```bash
pip install python-jose
```

## SAL

From the interactive shell:
```python
path = j.sal.fs.joinPaths('/', 'tmp', 'foo')
j.sal.fs.writeFile(path, 'hello world!')

j.sal.fs.readFile(path)
'hello world!'
```
## Prefab

```python
local_prefab = j.tools.prefab.local
local_prefab.network.zerotier.install()
local_prefab.network.zerotier.start()
zt_network_id = "17d709436c5bc232"
local_prefab.network.zerotier.network_join(network_id=zt_network_id)
zt_machine_addr = local_prefab.network.zerotier.get_zerotier_machine_address()
```

> See below for the authorization of the join request using the JumpScale client

## Config Manager

Initialize the Config Manager - from command line:
```bash
# Create a SSH key pair for encrypting the secret data
JS_CONFIG_SSHKEY_PATH="/root/.ssh/jsconfig_key"
ssh-keygen -t rsa -f $JS_CONFIG_SSHKEY_PATH -P ''

# Create a directory in j.dirs.CODEDIR for storing the config instances
GIT_SERVER="docs.grid.tf"
GIT_ACCOUNT="yves"
REPO_NAME="my_jsconfig"
JS_CONFIG_REPO_DIR="/opt/code/$GIT_SERVER/$GIT_ACCOUNT/$REPO_NAME"
mkdir -p $JS_CONFIG_REPO_DIR

# Optionally make this directory a Git repository
cd $JS_CONFIG_REPO_DIR
git init 

# Mark this directory as your config repository
js9_config init --path $JS_CONFIG_REPO_DIR --key $JS_CONFIG_SSHKEY_PATH
```

ItsYou.online:
```python
app_id = "g3XV1CMpTD2mvkaLqdtARmfbOASy"
secret = "6kRMy9Zzh99lsqDq6eyab6y_M5aN"
iyo_cfg = dict(application_id_=app_id, secret_= secret)

iyo_config_instance = j.tools.configmanager.configure(location="j.clients.itsyouonline", instance="main", data=iyo_cfg, interactive=True)
```

<a id="clients"></a>

## Clients

- [ItsYou.online](#iyo)
- [ZeroTier](#zt)
- [Packet.net](#packet)
- [OpenvCloud](#ovc)
- [Zero-OS](#zos)
- [SSH key](#sshkey)
- [SSH connection](#ss)

<a id="iyo"></a>

### ItsYou.online
```python
# First get the ItsYou.online client, from the previously 
# created config instance for ItsYou.online, in this case named 'main'
iyo_client = j.clients.itsyouonline.get(instance='main')

# Specify a scope
iyo_organization = 'zos-training-org'
memberof_scope = 'user:memberof:{}'.format(iyo_organization)

# Get the JWT
jwt = iyo_client.jwt_get(scope=memberof_scope, refreshable=True)
```


<a id="zt"></a>

### ZeroTier

You will need to add the ZeroTier module, later (also) needed by the Packet.net client:
```bash
pip install zerotier
```

```python
zt_network_id = '17d709436c5bc232'
zt_api_token = 'tFGYkdKutMR3crYA9FHfzVKKhMdbanDq'

zt_cfg = dict(token_=zt_api_token)

zt_cfg_instance_name = 'myzt'
zt_client = j.clients.zerotier.get(instance=zt_cfg_instance_name, data=zt_cfg)

zt_network = zt_client.network_get(network_id=zt_network_id)
```


Use the `zt_machine_addr` (see above) to authorize the join requestom from your machine:
```python
member = zt_network.member_get(address=zt_machine_addr)
```

Authorize the ZT join request:
```python
member.authorize()
```

<a id="packet"></a>

### Packet.net

Dependencies:
```bash
pip install packet-python
pip install zerotier
pip install ipcalc
```

If you already have a configuration instance for Packet.net:
```python
packet_cfg_instance_name = "yves"
packet_client = j.clients.packetnet.get(instance=packet_cfg_instance_name)
```

If not, create it:
```python
packet_api_key = "..."
packet_project_name = "GIG Engineering"
packet_cfg = dict(auth_token_=packet_api_key, project_name=packet_project_name)
packet_client = j.clients.packetnet.get(instance=packet_cfg_instance_name, data=packet_cfg)
```

Boot a Zero-OS node on Packet.net:
```python
zos_host_name = 'zos-training-node'
zos_kernel_params = ['organization={}'.format(iyo_organization), 'development']
zos_branch = 'development'

# Using packet_client.startZeroOS()
zos_client, zos_node, zt_ip_address = packet_client.startZeroOS(hostname=zos_host_name, plan='x1.small', facility='ams1', zerotierId=zt_network_id, zerotierAPI=zt_api_token, wait=True, remove=False, params=zos_kernel_params, branch=zos_branch)

# Or using packet_client.startDevice(), with an unsecure boot
ipxe_url = 'http://unsecure.bootstrap.gig.tech/ipxe/{}/{}/'.format(zos_branch, zt_network_id) + '%20'.join(zos_kernel_params)
zos_node = packet_client.startDevice(hostname=zos_host_name, plan='x1.small', facility='ams1', os='', ipxeUrl=ipxe_url)
```

In the last case we still need to authorize the ZT join request:
```python
# First get the public IP address
ipaddr = [netinfo['address'] for netinfo in zos_node.pubconfig['netinfo'] if netinfo['public'] and netinfo['address_family'] == 4]
public_ip = ipaddr[0]

zos_member = zt_network.member_get(public_ip=public_ip)
zos_member.authorize()
```

Getting the private IP address of the ZOS node:
```python
zt_ip_address = zos_member.private_ip
```

<a id="ovc"></a>

## OpenvCloud

Get a cloud space on an OpenvCloud environment:
```python
ovc_url = 'se-sto-en01-001.gig.tech'
ovc_location = 'se-gen-1'
ovc_instance_name = 'sweden'
ovc_cfg = dict(address=ovc_url, location=ovc_location)

ovc_config_instance = j.tools.configmanager.configure(location="j.clients.openvcloud", instance=ovc_instance_name, data=ovc_cfg)

ovc_client = j.clients.openvcloud.get(instance=ovc_instance_name)

ovc_account_name = 'yves'
vdc_name = 'zero-space'

ovc_account = ovc_client.account_get(name=ovc_account_name, create=False)
cloud_space = ovc_account.space_get(name=vdc_name, create=True)
#cloud_space = ovc_account.space_get(name=vdc_name, create=False)
```

Boot a VM with Zero-OS  in the cloud space:
```python
vm_name = 'zos-vm'
zos_kernel_params = ['organization={}'.format(iyo_organization), 'development']
zos_branch = 'development'

ipxe_url = 'ipxe: https://bootstrap.gig.tech/ipxe/{}/{}/'.format(zos_branch, zt_network_id) + '%20'.join(zos_kernel_params)
zos_vm = cloud_space.machine_create(name=vm_name, image='IPXE Boot', authorize_ssh=False, userdata=ipxe_url)
```

There is now a `userdata` argument in the `machines_create` Cloud API end point, will be available in the OVC client too, pass the iPXE value there, as indicated in https://github.com/0-complexity/openvcloud_ays/blob/master/_images/image_ipxe/readme.md


<a id="zos"></a>

## Zero-OS

```python
zos_cfg = dict(host=zt_ip_address, port=6379, password_=jwt)

zos_instance_name = 'node1'
#zos_cfg_instance = j.tools.configmanager.configure(location="j.clients.zero_os", instance=zos_instance_name, data=zos_cfg, interactive=True)
# zos_client = j.clients.zos.get(instance=zos_instance_name)
zos_client = j.clients.zos.get(instance=zos_instance_name, data=zos_cfg)
```

Get the SAL of the Zero-OS node:
```python
zos_node = j.clients.zero_os.sal.get_node(instance=zos_instance_name)
``` 

Create a container:
```python
your_zero_hub_account = 'yves'
flist_url = 'https://hub.gig.tech/{}/js9_sandbox_full.flist'.format(your_zero_hub_account)

js9_sandbox_container = zos_node.containers.create(name='js9_sandbox', flist=flist_url, hostname='js', nics=[{"type": "default"}], ports={2200: 22})

# Test
js9_sandbox_container.client.bash(script='. /env.sh; js9 "print(\'works in JS9!\')"').get()
```

Enable SSH access:
- [Create the SSH host keys](#)
...


```python
js9_sandbox_container.client.system(command='mkdir -p /etc/ssh/')
js9_sandbox_container.client.system(command='mkdir -p /var/run/sshd')
js9_sandbox_container.client.system(command='ssh-keygen -A')  
```

```python
script = "sudo sed -i 's/prohibit-password/without-password/' /etc/ssh/sshd_config"
js9_sandbox_container.client.bash(script=script).get()
```

```python
nft = zos_node.client.nft
nft.open_port(2200)
```

```python
job = js9_sandbox_container.client.system(command='sshd -D')
job.running
job.get()

```

<a id="sshkey"></a>

### SSH key


```python
sshkey_name = "another_key"
sshkey_path = "/root/.ssh/{}".format(sshkey_name)

sshkey_client = j.clients.sshkey.key_generate(path=sshkey_path, passphrase='hello')

# Check the result
!ls ~/.ssh
j.tools.configmanager.list("j.clients.sshkey")
```

Authorize key:]
```
js9_sandbox_container.client.system(command='touch /root/.ssh/authorized_keys')

public_key = j.clients.sshkey.get(instance=sshkey_name).pubkey

js9_sandbox_container.client.bash(script='echo \"{}\" >> \"/root/.ssh/authorized_keys\"'.format(public_key))

js9_sandbox_container.client.system(command='cat /root/.ssh/authorized_keys').get()
```


<a id="ssh"></a>

### SSH Connection

```python
# Get the private SSH key loaded by the SSH agent
sshkey_client.load()

node_ipaddr = "10.147.18.206"
sshconn_name = "js9_sandbox"
sshconn_cfg = dict(addr=node_ipaddr, port=2200, login="root", sshkey=sshkey_name, allow_agent=True)

ssh_client = j.clients.ssh.get(instance=ssh_conn_name, data=sshconn_cfg, use_paramiko=False)

ssh_client.isconnected
ssh_client.active
ssh_client.execute(cmd="hostname")
```

## Node Manager

```python
j.tools.nodemgr.set(name='js9_sandbox', sshclient=ssh_client_name, description="ssh connection to js9_sandbox in Zero-OS", clienttype="j.clients.ssh")
```
