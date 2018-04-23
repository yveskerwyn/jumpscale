# Happy Paths

- [Install JumpScale](#install-js)
- [Create a SSH key](#ssh-key)
- [Initialize the Configuration Manager](#init-config)
- [Creating a virtual machine ready for hosting Docker containers](#create-vm)


@ TODO
check:
```
js9_config list -d -u
```

<a id="install-js"></a>
## Install JS

See https://github.com/Jumpscale/bash/tree/development

```bash
curl https://raw.githubusercontent.com/Jumpscale/bash/development/install.sh?$RANDOM > /tmp/install.sh;bash /tmp/install.sh
source ~/.bashrc
export JS9BRANCH=development
ZInstall_host_js9_full
```

<a id="ssh-key"></a>
## Create SSH key

The JumpScale configuration manager will encrypt all secret configuration data with an SSH private key of choice.

In case you don't have any SSH key yet, create it - from the command line, with an empty passphrase:
```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa -P ''
```



Or using JumpScale:
```python
j.tools.prefab.local.system.ssh.keygen(user='root', name='id_rsa')
```

@TODO make sure to associate this key to your Git server account


<a id="init-config"></a>
## Initialize the Configuration Manager

At this point the JumpScale configuration manager is not yet initialized, as you can see in the JumpScale configuration file, `{j.dirs.HOSTCFGDIR}/cfg/jumpscale9.toml}`:
```toml
[myconfig]
giturl = ""
path = ""
sshkeyname = ""
```

Or from JumpScale:
```python
j.core.state.config_js["myconfig"]
{'giturl': '', 'path': '', 'sshkeyname': ''}
```

Options to initialize:
- From the command line:
    - using `js9_config check`
    - using `js9_config init` w/o options
    - using `js9_config init` w/ options
- From JumpScale

Using `js9_config check`:
- it will check for keys in ~/.ssh
- if no keys were found in ~/.ssh...
- if more then one is found it will ask which to use
- if only one found it will load that one in ssh-agent
- it will then ask to use loaded key as encryption key
- Git based or just local?
- if Git based, you need to specify Git URL, e.g. `ssh://git@docs.grid.tf:7022/yves/jsconfig.git`
- then you will be asked to fill out config data for `j.tools.myconfig`: `email`, `fullname` and `login_name`
- then it will ask to use chosen key to encrypt
- And finally you'll get:

configmanager:

- path: /opt/code/docs/yves/jsconfig
- keyname: id_rsa
- is sandbox: False
- sshagent loaded: True
- key in sshagent: False
```

See related issue: https://github.com/Jumpscale/bash/issues/76


In order to initialize execute `js9_config init` from the command line:
```bash
configpath="~/opt/code/github/yveskerwyn/jsconfig"
keypath="~/.ssh/id_rsa"
js9_config init --path $configpath --key $keypath
```

This will bring up two interactive screens:
- one for `j.clients.sshkey/id_rsa.toml`
- one for `j.tools.myconfig/main.toml`

Check the result:
```bash
js9_config check

configmanager:

- path: ~/opt/code/github/yveskerwyn/jsconfig
- keyname: id_rsa
- is sandbox: False
- sshagent loaded: True
- key in sshagent: True
```

Alternatively you can also use the `--silent` (`-s`) option in order to have JumpScale (silently) automatically figure our the configuration; in that case make sure you have the SSK key loaded...



Reset:
```bash
js9_config reset
```

This will:
- Delete all or only the specified configuration instances (specified with the optional `-l` and `-i` options)
- Reinitialize your configuration repository - allowing you to select/create a new SSH key that gets loaded by ssh-agent


JumpScale knows the location of the configuration repository by first checking for sandboxed secure configuration instances in your current directory, and if not found check the value of the `path` key in the `[myconfig]` section of your JumpScale configuration file, located in `{j.dirs.HOSTCFGDIR}/cfg/jumpscale9.toml}`. 

In case no value was set yet for the `path` key in the `[myconfig]` section of the JumpScale configuration file, JumpScale will walk over all Git repositories it finds under `$HOMEDIR/code/$type/$account/$reponame` and look for `.jsconfig`, and only in case it finds one it will set `path` value in the JumpScale configuration file to the repository where it found `.jsconfig`. In case no repository could be found, JumpScale will set the `path` value to the default `{j.dirs.CFGDIR}/myconfig`.

See [https://github.com/Jumpscale/core9/blob/development/docs/config/config_file_locations.md] for more details about this.

Optionally also reset the `path` value in the JumpScale configuration file - using JumpScale:
```python
#Don't use - this will wipe out jumpscale9.toml completelly: j.core.state.reset()
#In order to reset jumpscale9.toml to the default values use js9_init
# Check the current value
#j.core.state.config_js
j.core.state.config_js["myconfig"]
#j.core.state.config_js["myconfig"]["path"]
#j.core.state.configGetFromDict("myconfig", "path")
#j.core.state.config_js["myconfig"]["sshkeyname")
#j.core.state.configGetFromDict("myconfig", "sshkeyname")
#j.core.state.config_js["myconfig"]["giturl"]
#j.core.state.configGetFromDict("myconfig", "giturl")
j.core.state.configSetInDict("myconfig", "path", "")
j.core.state.configSetInDict("myconfig", "sshkeyname", "")
j.core.state.configSetInDict("myconfig", "giturl", "")
```



Create a new directory for your configuration repository and initialize it as a Git repository - from the command line:
```
my_js_config_directory="/opt/code/docs/yves/myconfig"
#rm -rf $my_js_config_directory
mkdir -p  $my_js_config_directory
cd  $my_js_config_directory
git init
```

> Note that in order for JumpScale to discover this configuration repository (in case not explicitly configured (yet) in jumpscale9.toml) needs to be created in a directory with following parent structure `$somewhere/code/$type/$account/$reponame`.

Then initialize the configuration repository - from the command line:
```bash
js9_config init --key ~/.ssh/id_rsa --path  $my_js_config_directory
```

```bash
#touch .jsconfig
#js9_config init
```

If you add the `-p` (`--path`) option the `.jsconfig` file is auto-created, and it also allows you initialize any Git repository, not just the current. In case you did not create the `.jsconfig` file and didn't specify the path to the configuration repository, JumpScale will check the value of the `path` key in the `[myconfig]` section of your the JumpScale configuration file (`{j.dirs.HOSTCFGDIR}/cfg`, where `{j.dirs.HOSTCFGDIR}` defaults to `~/js9host`), this value typically defaults to `{j.dirs.CFGDIR}/myconfig`; `{j.dirs.CFGDIR}` defaults to `/opt/`.

This will create two configuration instances,
- `j.clients.sshkey/id_rsa.toml`
    ````
    allow_agent = true

    duration = 86400

    passphrase_ = ''

    path = '/root/.ssh/id_rsa'

    privkey_ = ''

    pubkey = ''
    ```

- `j.tools.myconfig/main.toml`

    ```toml
    email = 'yves@gig.tech'

    fullname = 'Yves Kerwyn'

    login_name = 'yves'
    ```


first asking you to choose which SSH key you want to use - in case you have more than one SSH key in `~/.ssh/id_rsa`.

In case you have only one key you will be asked to confirm `(y/n)` that you want to use this key.

Or in order to run it silently, not popping up the interaction, use the `-s` (`--silent`) option in combination with the `-k` (`--key`) option:
```bash
js9_config init --silent --key ~/.ssh/id_rsa
```

In case you only have one key in `~/.ssh/` you can omit the `-k` option:
```bash
js9_config init -s
```

In order to initialize using JumpScale, first make sure `python-jose` is installed:
```bash
pip3 install python-jose
```

And here how to initialize the configuration repository:
```python
jsconfig = {}
jsconfig["email"] = "yves@gig.tech"
jsconfig["login_name"] = "yves"
jsconfig["fullname"] = "Yves Kerwyn"

ssh_key_path = "/root/.ssh/id_rsa"
config_path = "/opt/myconfig"

j.tools.configmanager.init(data=jsconfig, silent=True, configpath=config_path, keypath=ssh_key_path)
```

This will create two configuration instances:
- `/opt/myconfig/j.tools.myconfigmain.toml`: the configuration instance for the configuration manager itself  
- `/opt/myconfig/j.clients.sshkey/id_rsa.toml`: the configuration instance for the specified SSH key (`/root/.ssh/id_rsa`) that will be used for encrypting/decrypting all secret configuration data

As result also the `[myconfig]` section in `jumpscale9.toml` setting the value of to the `/opt/myconfig` configuration directory, and 


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
iyo_client = j.clients.itsyouonline.get(instance='main')
```

Or just use `j.clients.itsyouonline.get()`:
```python
iyo_client = j.clients.itsyouonline.get(instance='main', data=iyo_config, create=True, die=True, interactive=False)
```

Once you have the configuration instance for ItsYou.online, you can easily get a (refresheable) JWT:
```python
jwt = iyo_client.jwt_get(refreshable=True)
```

Do the same for OpenvCloud, prepare the configuration data for OpenvCloud:
```python
url = "ch-gen-1.gig.tech"
location = "ch-gen-1"
ovc_config = {
    "address": url,
    "location": location
}
```

Create an OpenvCloud configuration instance using `j.tools.configmanager.configure`:
```python
j.tools.configmanager.configure(location="j.clients.openvcloud", instance="swiss", data=ovc_config)
ovc_client = j.clients.openvcloud.get(instance="swiss")
```

Or using `j.clients.openvcloud.get()`:
```python
ovc_client = j.clients.openvcloud.get(instance="swiss", data=ovc_config, create=True, die=True, interactive=False)
```

Update config instance data:
```python
ovc_cfg = ovc_client.config
data_set(key="address", val="ch-gen-1.gig.tech", save=True)
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
vdc_name = "4wordpress2"
#location = ovc_client.locations[0]["name"]
account = ovc_client.account_get(name=account_name, create=False)
#cloud_space = account.space_get(name=vdc_name, create=True, location=location)
cloud_space = account.space_get(name=vdc_name, create=True)
```

Create a new SSH key:
```python
import os
ssh_key_name = "mytestkey1"
#ssh_key_passphrase ="hello"
ssh_key_path = os.path.expanduser("~/.ssh/{}".format(ssh_key_name))
#sshkey = j.clients.sshkey.key_generate(path=ssh_key_path, passphrase=ssh_key_passphrase, overwrite=False, load=True, returnObj=True)
sshkey = j.clients.sshkey.key_generate(path=ssh_key_path, overwrite=False, load=False, returnObj=True)
```


> The above will next to creating a new SSH key, also create a new `j.clients.sshkey` configuration instance
> By setting `load=True` the SSH key also gets loaded by the ssh-agent; which is not needed for creating a machine though

Check the newly created sshkey on disk:
```python
!ls ~/.ssh 
```

```python
machine_name = "4wordpress2"
machine = cloud_space.machine_get(name=machine_name, create=True, sshkeyname=ssh_key_name)
```

Check the node manager:
```python
node = j.tools.nodemgr.get(instance=machine_name, create=False)
node.config.data
```

Use Prefab:
```python
prefab = node.prefab

# make sure the apt-update/upgrade was done & some basic requirements
prefab.system.base.install()
prefab.virtualization.docker.install()
```

Check the (resulting new) SSH "connection" configuration instance that got created
```python
j.clients.ssh.list()
ssh_connection = j.clients.ssh.get(instance="185-15-201-118-2200-root-public")
ssh_connection
```

++++
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


+++

List all nodes:
```bash
js9_node list
```


SSH into a machine; this requires that the ssh key is loaded:
```bash
js9_node ssh -i proxy
```





to reset:
```bash
rm -rf /opt/code/docs/yves/jsconfig
j.core.state.configSetInDict("myconfig", "path", "")
j.core.state.configSetInDict("myconfig", "sshkeyname", "")
j.core.state.configSetInDict("myconfig", "giturl", "")
```
