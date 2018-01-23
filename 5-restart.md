# Restart AYS Server

When rebooting any of the three hosts you'll need to restart the tmux sessions.

Below we discuss how to do that from the JumpScale interactive shell:
- [From the interactive shell on the hosts](#locally)
- [From a remote interactive shell](#remotely)

<a id="locally"></a>
## Locally on the hosts

On the AYS server host execute:
```python
j.tools.prefab.local.js9.atyourservice.start(host="0.0.0.0")
```

On the AYS portal host execute:
```python
j.tools.prefab.local.web.portal.start()
```

On the Caddy host execute:
```python
j.tools.prefab.local.web.caddy.start()
```

<a id="remotely"></a>
## From a remote interactive shell

Make sure you have your ItsYou.online application ID and secret available, for instance from environment variables:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
```

Get the OpenvCloud details of the hosts:
```python
ovc_url = "be-gen-1.demo.greenitglobe.com"
account_name = "Account_of_Yves"
cloud_space_name = "ays-space"
location = "be-gen-1"
ays_host_name = "ays-server"
portal_host_name = "portal"
caddy_host_name = "caddy"
```

Connect to the cloud space using the JumpScale client for OpenvCloud:
```python
ovc_client = j.clients.openvcloud.get(app_id, secret, ovc_url)
account = ovc_client.account_get(account_name, create=False) 
vdc = account.space_get(cloud_space_name, location)
```

Make sure you have private SSH key loaded that was used to setup the hosts:
```python
key_path = os.path.expanduser("~/.ssh/mykey")
j.clients.ssh.load_ssh_key(path=key_path, create_keys=True)
j.clients.ssh.ssh_keys_list_from_agent()
```

Connect to the AYS server host:
```python
ays_host = vdc.machine_get(name=ays_host_name, create=False)
```

Start the AYS server:
```python
ays_host.prefab.js9.atyourservice.start(host="0.0.0.0")
```

Start the AYS Portal:
```python
j.tools.prefab.local.web.portal.start()
```

Start the Caddy:
```python
j.tools.prefab.local.web.caddy.start()
```