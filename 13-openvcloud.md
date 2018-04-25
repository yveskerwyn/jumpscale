# Interacting with OpenvCloud

Make sure you have initialized the JumpScale Configuration Manager as documented in [Getting started with the Config Manager ](11-config_manager_getting_started.md).

Steps:
- [Create a config instance for OpenvCloud](#config-instance)
- [Create a cloud space](#cloud-space)
- [Create a virtual machine](#vm)

<a id="config-instance"></a>

## Create a config instance for OpenvCloud

Creating an OpenvCloud configuration instance can be done interactively:
```python
j.tools.configmanager.configure(location="j.clients.openvcloud", instance="main")
```

Or using the bash tool:
```bash
js9_config configure -l j.clients.openvcloud -i main
```

Based on this configuration instance of OpenvCloud you can get an OpenvCloud client:
```python
ovc_client = j.clients.openvcloud.get(instance="main")
```

> The above will automatically look for the `main` ItsYou.online configuration instance in order get a JWT needed to connect to OpenvCloud.

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

<a id="cloud-space"></a>

## Create a cloud space

Create a cloud space:
```python
account_name = "Account_of_Yves"
vdc_name = "delete-me"
location = ovc_client.locations[0]["name"]
account = ovc_client.account_get(name=account_name, create=False)
cloud_space = account.space_get(name=vdc_name, create=True, location=location)
```

<a id="vm"></a>

## Create a virtual machine

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

Check the result:
```python
j.clients.sshkey.list()
```

Or for an already existing SSH key:
```python
my_sshkey =  j.clients.sshkey.get(instance='example_key', data=dict(path='/root/.ssh/id_rsa'))
```
