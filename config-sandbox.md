# Configuration Sandbox

See:
- https://github.com/Jumpscale/core9/blob/development/docs/config/config_file_locations.md
- https://github.com/Jumpscale/core9/blob/development/docs/config/configmanager.md


Make sure you have the `secureconfig` and `key` directories.

Add the following to a new module `base.py` under a module directory `lib`'

```python
def node_get(name, sshkeyname="", reset=False):

    #make sure we have logging
    j.logger.logger_filters_add("prefab,exec,ssh,*")

    # this makes sure that everything is configured in this sandbox, e.g. sshkey loaded, the local config dir used
    # import ipdb; ipdb.set_trace()
    sshkey = j.tools.configmanager.sandbox_check()
    ovccl = j.clients.openvcloud.get(instance="swiss")
    location = ovccl.config.data["location"]
    account = ovccl.account_get(account=account_name, create=True)
    space = account.space_get(name=vdc_name, create=True)

    if not sshkeyname:
        sshkeyname = sshkey.instance

    if not sshkeyname:
        raise RuntimeError("need sshkeyname")

    machine = space.machine_get(name=vdc_name, reset=reset, create=True, sshkeyname=sshkeyname)

    n = j.tools.nodemgr.get(name, create=False)
    p = n.prefab

    # make sure the apt-update/upgradre was done & some basic requirements
    p.system.base.install()

    return machine, n
```


```python
from js9 import j
from lib.base import node_get

new_vm_name = "delete-me"
machine, node = node_get(name=new_vm_name)
```