# Getting started with the Config Manager 

See: https://github.com/Jumpscale/core9/blob/development/docs/config/configmanager.md




## Start from an existing config repository

In order to check whether there is already a config repository:
```python
j.tools.configmanager._findConfigRepo()
```



Working with the JumpScale config manager requires a private SSH key, which will be used by JumpScale to decrypt secret configuration data.

The following will list your ssh **connections** that are configured on your in the JumpScale config manager on the local machine:
```python
j.clients.ssh.list()
```

Or:
```python
j.clients.ssh.getall()
```

Data returned per sshkey
```python
addr = '185.15.201.106'

addr_priv = '192.168.103.252'

allow_agent = true

clienttype = ''

forward_agent = true

login = 'root'

passwd_ = ''

port = 2202

port_priv = 22

proxy = ''

sshkey = 'mykey'

stdout = true
```

In order to delete entries:
```bash
j.clients.ssh.delete(instance='185-15-201-106-2000-root-public')
```

> note: `j.clients.ssh.list()` lists all SSH keys that are in the config manager
> while ` `j.clients.sshkey.list()` lists all SSH keys that are loaded by the ssh-agent


Also see:
```bash
ls j.clients.sshkey -1
id_rsa.toml
mykey.toml
mykey2.toml
```

And:
```bash
ls j.clients.ssh -1
185-15-201-106-2200-root-public.toml
185-15-201-106-2201-root-public.toml
185-15-201-106-2202-root-public.toml
185-15-201-106-2203-root-public.toml
```

Checking the `sshkey`configuration data:
```bash
cat j.clients.sshkey/id_rsa.toml
```

Output:
```bash
allow_agent = true

duration = 86400

passphrase_ = ''

path = ''

privkey_ = ''

pubkey = ''
```

Checking the `ssh` configuration data:
```bash
cat j.clients.ssh/185-15-201-106-2200-root-public.toml
```

Output:
```bash
addr = '185.15.201.106'

addr_priv = '192.168.103.254'

allow_agent = true

clienttype = ''

forward_agent = true

login = 'root'

passwd_ = ''

port = 2200

port_priv = 22

proxy = ''

sshkey = 'mykey'

stdout = true

timeout = 300
```


In case you didn't yet initialize a config repository it will try to do so. For this JumpScale needs to know which SSH key you want to use the encrypt the secret data in the config repository. Which is a bit a chicken and egg situation. At this point it check the ssh-agent, and invited you to select the key you want JumpScale to use for your new config repository.

In the case that no key was loaded, it will check for keys in `/root/.ssh/`. and invite you to select one of the found keys, or in case there was only one found it will propose to use that one.


Or in the case that there is no ssh-agent running yet, it will first start an ssh agent and then check for keys in `/root/.ssh/`. If only one found it will propose to use that one:
```python
In [1]: j.clients.ssh.list()
* Will start agent
* load ssh agent
* load ssh key: /root/.ssh/id_rsa
Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)
Lifetime set to 86400 seconds
* JS9 init: ssh key found in agent is:'/root/.ssh/id_rsa'
* JS9 init: Is it ok to use this one:'/root/.ssh/id_rsa'?
 [y/n]:
 ```

 > In the special case that you have a forwarded keys loaded by ssh agent, it will also include that key as a possible key for you to choose for the new config repository. Choosing this key is currently however untested/unsupported:
```python
* JS9 init: ssh key found in agent is:'/Users/yves/.ssh/id_rsa'
* JS9 init: Is it ok to use this one:'/Users/yves/.ssh/id_rsa'?
```


Also see: https://github.com/Jumpscale/0-robot/blob/master/utils/scripts/packages/dockerentrypoint.py
```python
if not j.sal.fs.exists("/root/.ssh/id_rsa"):
    j.tools.prefab.local.system.ssh.keygen(user='root', name='id_rsa')

j.tools.prefab.local.core.run("js9_config init --silent --path /tmp/myconfig --key /root/.ssh/id_rsa")
```

See the options:
```bash
root@vm-913:/opt/cfg# js9_config init --help
Usage: js9_config init [OPTIONS]

  js9_config init -s

Options:
  -s, --silent     if silent will try to figure out configuration
                   automatically, make sure 1 sshkey loaded in ssh-agent.
  -p, --path TEXT  path of the configuration repository you want to use
  -k, --key TEXT   path to the ssh key you want to use
  --help           Show this message and exit.
  ```

Get the path the exiting config repository:
```python
j.tools.configmanager.path_configrepo
```



```python
j.tools.myconfig
```



## Create a new config repository

If not already done, we need to create a configuration instance for ItsYou.online. This requires a Git repository that was initialized as the JumpScale configuration repository:
```bash
js9_config init
```

Or from the interactive shell:
```python
j.tools.configmanager.init()
```

In the above case, you're not passing any arguments, as a result you'll get answer some questions:
```python
In [1]: j.tools.configmanager.init()
* JS9 init: ssh key found in agent is:'/Users/yves/.ssh/id_rsa'
* JS9 init: Is it ok to use this one:'/Users/yves/.ssh/id_rsa'?
 [y/n]: y
* JS9 init: Do you want to use a git based CONFIG dir, y/n?
 [y/n]: n
* JS9 init: will create config dir in '/opt/cfg/myconfig/', your config will not be centralised! Is this ok?
 [y/n]: y
```

```python
jsconfig = {}
jsconfig["email"] = "yves@gig.tech"
jsconfig["login_name"] = "yves"
jsconfig["fullname"] = "Yves Kerwyn"

ssh_key_path = "/root/.ssh/id_rsa"
config_path = "/tmp/yveskerwyn"


j.tools.configmanager.init(data=jsconfig, silent=False, configpath=config_path, keypath=ssh_key_path)
```

Check:
```python
j.tools.configmanager.path
```

```bash
/tmp/yveskerwyn/j.clients.sshkey/id_rsa.toml
allow_agent = true

duration = 86400

passphrase_ = ''

path = ''

privkey_ = ''

pubkey = ''
```

Also see the `~/js9host/cfg/jumpscale9.toml`: 
```toml
[myconfig]
sshkeyname = "id_rsa"
path = "/tmp/yveskerwyn"
giturl = ""
```

Also to check:
```python
j.tools.myconfig
```


Adding a JumpScale configuration instance for ItsYou.online is done as follows:
```bash
js9_config configure -l j.clients.itsyouonline -i main -s /root/.ssh/id_rsa
```

This will bring up a configuration form where you can fill out the ItsYou.online `baseurl`, your `application ID` and `secret`.

In JumpScale you can list all ItsYou.online configuration instances as follows:
```python
j.tools.configmanager.list(location="j.clients.itsyouonline")
```

> Note that currently only one ('main') configuration instance can exists for ItsYou.online

To interactively update a configuration instance:
```python
j.tools.configmanager.configure(location="j.clients.itsyouonline", instance="main")
```

Or w/o interaction:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
iyo_config = {
    "application_id_": app_id,
    "secret_": secret
}
#j.tools.configmanager.configure(location="j.clients.itsyouonline", data=iyo_config, instance="main", sshkey_path="/root/.ssh/bootstrap_vm2_key")
j.tools.configmanager.configure(location="j.clients.itsyouonline", data=iyo_config, instance="main")
```

Based on this ('main') configuration instance you then can get an ItsYou.online client as follows:
```python
iyo_client = j.clients.itsyouonline.get(instance="main")
```

By also passing the `application ID` and `secret` through the `data` argument you can override (and update) the ItsYou.online configuration instance that is used for connecting ItsYou.online:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
iyo_config = {
    "application_id_": app_id,
    "secret_": secret
}

iyo_client = j.clients.itsyouonline.get(instance="main", data=iyo_config)
```

Next you will need to set OpenvCloud configuration instance, this can be done interactively:
```python
j.tools.configmanager.configure(location="j.clients.openvcloud", instance="main")
```

Or using the bash tool:
```bash
js9_config configure -l j.clients.openvcloud -i main --silent --key /root/.ssh/id_rsa
```

Based on this configuration you can get an OpenvCloud client:
```python
ovc_client = j.clients.openvcloud.get(instance="main")
```

> The above will automatically look for the `main` ItsYou.online configuration instance in order get a JWT needed to connect to OpenvCloud.

The above will bring up an interactive screen asking you which ssh key you want to use for decrypting the encrypted configuration information. 

You can pass the path to the private key upfront as follows - not true anymore:
```python
#import os
#ssh_key_name = "bootstrap_vm2_key"
#ssh_key_name = "id_rsa"
#key_path = os.path.expanduser("~/.ssh/{}".format(ssh_key_name))
ovc_client = j.clients.openvcloud.get(instance="main")
```

Just as with the ItsYou.online client you can override (update) configuration instance, or even create a new instance with different data:
```python
url = "ch-gen-1.demo.greenitglobe.com"
location = "ch-gen-1"
ovc_config = {
    "address": url,
    "location": location
}

ovc_client = j.clients.openvcloud.get(instance="swiss", data=ovc_config)
```

As a result of the above you will now have two OpenvCloud configuration instances, of which the newly created one `swiss` is used:
```python
j.tools.configmanager.list("j.clients.openvcloud")
```

Since the OpenvCloud client factory (`OVCClientFactory`) inherits from `JSConfigBaseFactory` there is also a `list()` method to list all available configuration instances for the OpenvCloud client as follows: 
```python
j.clients.openvcloud.list()
```

You can update the newly created OpenvCloud configuration instance as follows:
```python
#j.tools.configmanager.configure(location="j.clients.openvcloud", instance="swiss", data=ovc_config, sshkey_path=key_path)
j.tools.configmanager.configure(location="j.clients.openvcloud", instance="swiss", data=ovc_config)
```

Instead of using the JWT from the ItsYou.online configuration instance, you can also pass a JWT explicitly, and have it saved to a (new) OpenvCloud configuration instance:

```python
data = {
    'address': 'be-gen-1.demo.greenitglobe.com',
    'port': 443,
    'appkey_': 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJlMnpsTi03U0M2N3RhdjN0UlJuZG9VQUd4a1U1IiwiZXhwIjoxNTE4NzEyOTE4LCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieXZlcyJ9._oLfHc_WDHaToo26NJEOnBDliQncWBtlYDO3doLGf0V2lCoXdCST-FdJYm5TIGbvpOL5_B6cVXriIS_ctTuKTZaKNhdPtX2Jhc1T2whiEt8_Q-CwJgzTWwUiL9oHMAeQ'
}
ovc_client = j.clients.openvcloud.get(instance='belgium', data=data)
```

Of course you can also omit the `instance` argument, and simply pass the data:
```python
data = {
    'address': 'be-gen-1.demo.greenitglobe.com',
    'port': 443,
    'appkey_': 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJlMnpsTi03U0M2N3RhdjN0UlJuZG9VQUd4a1U1IiwiZXhwIjoxNTE4NzEyOTE4LCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieXZlcyJ9._oLfHc_WDHaToo26NJEOnBDliQncWBtlYDO3doLGf0V2lCoXdCST-FdJYm5TIGbvpOL5_B6cVXriIS_ctTuKTZaKNhdPtX2Jhc1T2whiEt8_Q-CwJgzTWwUiL9oHMAeQ'
}
ovc_client = j.clients.openvcloud.get(data=data)
```

Create a cloud space:
```python
account_name = "Account_of_Yves"
vdc_name = "delete-me"
location = ovc_client.locations[0]["name"]
account = ovc_client.account_get(name=account_name, create=False)
cloud_space = account.space_get(name=vdc_name, create=True, location=location)
```

Next we want to create a virtual machine, which requires a SSH key. Let's first create a new one, here we do it using `j.tools.prefab.local.system.ssh.keygen`, better would be to use `j.clients.sshkey.get()` as shown later:
```python
ssh_key_name2 = "mykey"
j.tools.prefab.local.system.ssh.keygen(user='root', name=ssh_key_name2)
key_path2 = os.path.expanduser("~/.ssh/{}".format(ssh_key_name2))
```

Check the result:
```python
!ls ~/.ssh
```

From Reem, instead of using `j.tools.prefab.local.system.ssh.keygen()` use `j.clients.sshkey.key_generate()`, this will create a config instance, which is important in order to create a VM w/o having to get the interactive screen that you get in case you work with `ssh.keygen()`:
```python
import os
ssh_test_key1_name = "mytestkey1"
ssh_test_key1_path = os.path.expanduser("~/.ssh/{}".format(ssh_test_key1_name))
#j.clients.sshkey.key_generate('{home}/.ssh/test')
j.clients.sshkey.key_generate(path=ssh_test_key1_path, passphrase='hello', overwrite=False, load=False, returnObj=True)
```








And now let's create a SSH key configuration instance for the newly created SSH key - doesn't work yet, and not needed for creating a machine:
```python
ssh_key_config = {}
ssh_key_config["allow_agent"] = True
ssh_key_config["duration"] = 86400
ssh_key_config["path"] = key_path2
....

#j.tools.configmanager.configure(location="j.clients.sshkey", instance="mykey", data=data, sshkey_path=key_path2)
ssh_key2 = j.clients.sshkey.get(instance=ssh_key_name2, data=ssh_key_config, create=False, die=True, interactive=False)
```

Create a new VM:
```python
machine_name = "delete-me"
machine = cloud_space.machine_get(name=machine_name, create=True, sshkeyname=ssh_key_name2)
```

From Reem:
```python
def node_get(name, sshkeyname="", reset=False):

    if not sshkeyname:
        sshkeyname = sshkey.instance

    if not sshkeyname:
        raise RuntimeError("need sshkeyname")

    machine = space.machine_get(name, reset=reset, create=True, sshkeyname=sshkeyname)

    n = j.tools.nodemgr.get(name, create=False)
    p = n.prefab

    # make sure the apt-update/upgradre was done & some basic requirements
    p.system.base.install()

    return machine, n
```




If not explicitely done upfront, The above will bring up the Configuration Manager interaction, forcing you to create a config instance for the new key, asking for:
- allow_Agent
- passphrase_
- privkey_
- path
- duration
- pubkey




Get node from nodemgr:
```python
node = j.tools.nodemgr.get(instance=machine_name, create=False)
prefab = node.prefab
```

Check:
```python
node.config.data
```

Install Docker on the newly created machine:
```python
prefab.virtualization.docker.install()
```

