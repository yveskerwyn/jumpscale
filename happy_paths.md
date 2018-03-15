# Happy Paths

- [Starting from zero](#from-zero)
- [Creating a virtual machine ready for hosting Docker containers](#create-vm)


<a id="from-zero"></a>
## Starting from zero

Reset:
```bash
js9_config reset
```


```python
jsconfig = {}
jsconfig["email"] = "yves@gig.tech"
jsconfig["login_name"] = "yves"
jsconfig["fullname"] = "Yves Kerwyn"

ssh_key_path = "/root/.ssh/id_rsa"
config_path = "/opt/myconfig"

j.tools.configmanager.init(data=jsconfig, silent=False, configpath=config_path, keypath=ssh_key_path)
```

This will create:
- `main.toml` in `/opt/myconfig`: a configuration repository 
- `id_rsa.toml` in `/opt/myconfi/j.clients.sshkey`: a configuration instance for the specified SSH key (`/root/.ssh/id_rsa`) that will be used for encrypting all secret configuration data 
- a configuration instance in `/opt/myconfij.tools.myconfig` 


You can also update the `[myconfig]` section in `jumpscale9.toml` as follows, here changing the name of the SSH key to use for the encryption:
```python
j.core.state.configSetInDict("myconfig", "sshkeyname", "id_rsa")
```

Or check the value as follows:
```python
j.core.state.configGetFromDict("myconfig", "sshkeyname") 
```

Prepare the configuration data for a new ItsYou.online configuration instance:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
iyo_config = {
    "application_id_": app_id,
    "secret_": secret
}
```

Create a configuration instance for ItsYou.online, using `j.tools.configmanager.configure`:
```python
j.tools.configmanager.configure(location="j.clients.itsyouonline", instance="main", data=iyo_config)
```

Or using `j.clients.itsyouonline.get()`:
```python
iyo_client = j.clients.itsyouonline.get(instance='main', data=iyo_config, create=True, die=True, interactive=False)
```

Do the same for OpenvCloud, prepare the configuration data for OpenvCloud:
```python
url = "ch-gen-1.demo.greenitglobe.com"
location = "ch-gen-1"
ovc_config = {
    "address": url,
    "location": location
}
```

Create an OpenvCloud configuration instance using `j.tools.configmanager.configure`:
```python
j.tools.configmanager.configure(location="j.clients.openvcloud", instance="swiss", data=ovc_config)
```

Or using `j.clients.openvcloud.get()`:
```python
ovc_client = j.clients.openvcloud.get(instance="swiss", data=ovc_config, create=True, die=True, interactive=False)
```

<a id="create-vm"></a>
## Creating a virtual machine ready for hosting Docker containers

Get the configuration instance for ItsYou.online and OpenvCloud:
```python
iyo_client = j.clients.itsyouonline.get(instance="main")
ovc_client = j.clients.openvcloud.get(instance="swiss")
```

Create a cloud space:
```python
account_name = "Account_of_Yves"
vdc_name = "delete-me"
#location = ovc_client.locations[0]["name"]
account = ovc_client.account_get(name=account_name, create=False)
#cloud_space = account.space_get(name=vdc_name, create=True, location=location)
cloud_space = account.space_get(name=vdc_name, create=True)
```

Create a new SSH key:
```python
import os
ssh_key_name = "mytestkey1"
ssh_key_passphrase ='hello'
ssh_key_path = os.path.expanduser("~/.ssh/{}".format(ssh_key_name))
sshkey = j.clients.sshkey.key_generate(path=ssh_key_path, passphrase=ssh_key_passphrase, overwrite=False, load=True, returnObj=True)
```

> The above will next to creating a new SSH key, also create a new `j.clients.sshkey` configuration instance, but only if `load=True`.
> In case `load=False`, it is not clear how you can still create an `j.clients.sshkey` configuration instance...

Check the newly created sshkey on disk:
```python
!ls ~/.ssh 
```

```python
machine_name = "delete-me"
machine = cloud_space.machine_get(name=machine_name, create=True, sshkeyname=ssh_key_name)
```

Check the node manager:
```python
node = j.tools.nodemgr.get(name, create=False)
node.config.data
```

Use Prefab:
```python
prefab = node.prefab

# make sure the apt-update/upgrade was done & some basic requirements
prefab.system.base.install()
prefab.docker.install()
```

Check the (resulting new) SSH "connection" configuration instance that got created
```python
j.clients.ssh.list()
ssh_connection = j.clients.get(instance="")
ssh_connection
```

Normally you would never create an SSH "connection" configuration instances explicitly, they are automatically created when connecting to nodes, but you can also create them explicitly, as follows, first prepare the configuration data:
```python
ssh_connection_config = {
    addr = '195.134.212.32'
    allow_agent = true
    forward_agent = true    
    login = 'root'    
    port = 2208    
    sshkey = 'itenv_test'
}
```

Or:
```python
ssh_connection_config = {
    login = "cloudscalers"
    passwd = "***"
    addr = '195.134.212.32'
}
```

Create the SSH "connection" configuration instance:
```python
ssh_connection = j.clients.ssh.get(instance='my-ssh-connection', data=ssh_connection_config, create=True, die=True, interactive=False)
```

