# Getting started with the Config Manager 

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
data = {
    "application_id_": app_id,
    "secret_": secret
}

iyo_client = j.clients.itsyouonline.get(instance="main", data=data)
```

Next you will need to set OpenvCloud configuration instance, this can be done interactively:
```
j.tools.configmanager.configure(location="j.clients.openvcloud", instance="main")
```

Or using the bash tool:
```bash
js9_config configure -l j.clients.openvcloud -i main -s /root/.ssh/id_rsa
```

Based on this configuration you can get an OpenvCloud client:
```python
ovc_client = j.clients.openvcloud.get(instance="main")
```

> The above will automatically look for the `main` ItsYou.online configuration instance in order get a JWT needed to connect to OpenvCloud.

The above will bring up an interactive screen asking you which ssh key you want to use for decrypting the encrypted configuration information. 


You can pass the path to the private key upfront as follows:
```python
import os
ssh_key_name = "bootstrap_vm2_key"
key_path = os.path.expanduser("~/.ssh/{}".format(ssh_key_name))
ovc_client = j.clients.openvcloud.get(instance="main", sshkey_path=key_path)
```

Just as with the ItsYou.online client you can override (update) configuration instance, or even create a new instance with different data:
```python
url = "ch-gen-1.demo.greenitglobe.com"
location = "ch-gen-1"
ovc_config = {
    "address": url,
    "location": location
}

ovc_client = j.clients.openvcloud.get(instance="swiss", data=ovc_config, sshkey_path=key_path)
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
ovc_client = j.clients.openvcloud.get(instance='belgium', data=data, sshkey_path=key_path)
```

Of course you can also omit the `instance` argument, and simply pass the data:
```python
data = {
    'address': 'be-gen-1.demo.greenitglobe.com',
    'port': 443,
    'appkey_': 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJlMnpsTi03U0M2N3RhdjN0UlJuZG9VQUd4a1U1IiwiZXhwIjoxNTE4NzEyOTE4LCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieXZlcyJ9._oLfHc_WDHaToo26NJEOnBDliQncWBtlYDO3doLGf0V2lCoXdCST-FdJYm5TIGbvpOL5_B6cVXriIS_ctTuKTZaKNhdPtX2Jhc1T2whiEt8_Q-CwJgzTWwUiL9oHMAeQ'
}
ovc_client = j.clients.openvcloud.get(data=data, sshkey_path=key_path)
```

Create a cloud space:
```python
account_name = "Account_of_Yves"
vdc_name = "delete-me"
location = ovc_client.locations[0]["name"]
account = ovc_client.account_get(name=account_name, create=False)
cloud_space = account.space_get(name=vdc_name, create=True, location=location)
```

Next we want to create a virtual machine, which requires a SSH key. Let's first create a new one:
```python
ssh_key_name2 = "mykey"
key_path2 = os.path.expanduser("~/.ssh/{}".format(ssh_key_name2))
j.clients.ssh.load_ssh_key(path=key_path2, create_keys=True)
```

And now let's create a SSH key configuration instance for the newly created SSH key - doesn't work yet:
```python
data = {}
data["..."] = ...
data["..."] = key_path2
data["..."] = ...

....

j.tools.configmanager.configure(location="j.clients.sshkey", instance="mykey", data=data, sshkey_path=key_path)

```

Check the loaded SSH keys - will not work anymore once 9.3.0_ssh got merged:
```python
j.clients.ssh.ssh_keys_list_from_agent()
```

Add a key - only works on 9.3.0_ssh:
```python
sshkey_name = "bootstrap_vm2_key"
sshkey = j.clients.sshkey.get(instance=sshkey_name)
sshkey.load()
```

Check the updated instance in the configmanager - only works on 9.3.0_ssh:
```python
j.tools.configmanager.list(location="j.clients.ssh")
```

Get one of them - doesn't work yet on development branch:
```python
import os
sshkey = j.tools.configmanager.get(location="j.clients.sshkey", instance=sshkey_name)
sshkey_name = sshkey.instance
key_path = os.path.expanduser("~/.ssh/{}".format(sshkey_name))
```

Create a new VM:
```python
machine_name = "delete-me"
machine = cloud_space.machine_get(name=machine_name, create=True, sshkeyname=ssh_key_name2)
#machine = cloud_space.machine_get(name=machine_name, reset=True, create=True, sshkeyname=ssh_key_name2)
```

Get node from nodemgr:
```python
node = j.tools.nodemgr.get(instance=machine_name, create=False)
prefab = node.prefab
```

