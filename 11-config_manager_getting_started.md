# Getting started with the Config Manager 

See: https://github.com/Jumpscale/core9/blob/development/docs/config/configmanager.md


## Initialize the configuration manager

Steps:
- [Create the SSH key](#ssh-key)
- [Create Git repository](#git-repo)
- [Initialize the config manager](#init)


<a id="ssh-key"></a>

### Create the SSH key

The JumpScale configuration manager will encrypt all secret configuration data with an SSH private key of choice.

In case you don't have any SSH key yet, create it from the command line, here with an empty passphrase:
```bash
export JS_CONFIG_SSHKEY_NAME="~/.ssh/id_rsa"
ssh-keygen -t rsa -f ~/.ssh/$JS_CONFIG_SSHKEY_NAME -P ''
```

Or using JumpScale:
```python
import os
ssh_key_name = os.environ["JS_CONFIG_SSHKEY_NAME"]
j.tools.prefab.local.system.ssh.keygen(user='root', keytype='rsa', name=ssh_key_name)
```


<a id="git-repo"></a>

### Create Git repository

Create a new Git repository under `j.dirs.CODEDIR`, which typically is `/opt/code`.

We recommend to create the repository in the following path: `$GIT_SERVER/$GIT_ACCOUNT/$REPO_NAME`:
```bash
GIT_SERVER="docs.grid.tf"
GIT_ACCOUNT="yves"
REPO_NAME="my_jsconfig"
JS_CONFIG_REPO_DIR="/opt/code/$GIT_SERVER/$GIT_ACCOUNT/$REPO_NAME"
mkdir -p $JS_CONFIG_REPO_DIR
cd $JS_CONFIG_REPO_DIR
git init
```

<a id="git-repo"></a>

### Initialize the config manager

In order to mark the above created Git repository as your configuration repository execute:
```bash
js9_config init --path $JS_CONFIG_REPO_DIR --key $JS_CONFIG_SSHKEY_PATH
``` 

Or from the interactive shell:
```python
sshkey_name = "id_rsa"
sshkey_path = "/root/.ssh/{}".format(sshkey_name)

git_server="docs.grid.tf"
git_account="yves"
repo_name="my_jsconfig"
config_path = "{}/{}/{}/{}".fornmat(j.dirs.CODEDIR, git_server, git_account, repo_name)

jsconfig = {}
jsconfig["email"] = "yves@gig.tech"
jsconfig["login_name"] = "yves"
jsconfig["fullname"] = "Yves Kerwyn"

j.tools.configmanager.init(data=jsconfig, silent=False, configpath=config_path, keypath=sshkey_path)
```

This will pop up to interactive screens:
- the first one to collect configuration data for `j.clients.sshkey`
- the second one to collect configuration data for `j.tools.myconfig`

The first one should not be filled out, since all data come from the `$JS_CONFIG_SSHKEY_PATH` you passed:

![](j.clients.sshkey.png)


In the second you can specify your e-mail address, full name and your (ItsYou.online or other) username:

![](j.tools.myconfig.png)


As a result two configuration files will have been created:
- `$JS_CONFIG_REPO_DIR/j.clients.sshkey/id_rsa.toml`
- `$JS_CONFIG_REPO_DIR/j.tools.myconfig/main.toml`



In order to check whether there is already a config repository:
```python
j.tools.configmanager._findConfigRepo()
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

