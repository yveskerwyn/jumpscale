## AYS Setup

Steps:
- [Variables](#vars)
- [Create the host](#create-host)
- [Authorize your pubic SSH key](#add-key)
- [Install JumpScale](#install-jumpscale)
- [Install AYS Server](#install-ays)
- [Start and test the AYS server](#start-ays)
- [Add the AYS API console](#api-console)
- [Add JumpScale Portal](#add-portal)
- [Add the AYS Portal App](#add-cockpit)
- [Configure the AYS Portal App](#configure-cockpit)
- [Add ItsYou.online integration](#add-iyo)
- [Add Caddy](#caddy)


<a id="vars"></a>
##  Variables

```python
import os
import yaml
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
git_token = os.environ["GITHUB_PAT"]
key_path = os.path.expanduser("~/.ssh/mykey")
ovc_url = "se-gen-1.demo.greenitglobe.com"
location = "se-gen-1"
account_name = "Account_of_Yves"
cloud_space_name = "ays-space"
ays_host_name = "ays-server"
portal_host_name = "portal"
pubkey_label = "default"
test_repo_name = "test_repo"
test_account_name = "myaccount"
test_vdc_name = "myvdc"
test_vm_name = "testvm"
ays_clients_org_name = "ays-server-clients-org"
api_key_label = "ays-server-api-key"
portal_users_org_name = "ays-portal-users"
branch = "9.2.1"
ays_templates_branch="9.2.1"
caddy_host_name = "caddy"
```

<a id="create-host"></a>
##  Create the host

Access your OVC account:
```python
ovc_client = j.clients.openvcloud.get(app_id, secret, ovc_url)
account = ovc_client.account_get(account_name, create=False) 
```

Get or create the VDC:
```python
vdc = account.space_get(cloud_space_name, location)
```

Make sure "mykey" is loaded:
```python
j.clients.ssh.ssh_keys_list_from_agent()
j.clients.ssh.load_ssh_key(path=key_path, create_keys=True)
```

If not yet created, create the host:
```python
ays_host = vdc.machine_get(name=ays_host_name, create=True, sshkeyname="mykey", sizeId=3)
```

Get the internal IP address of the VM - will be used later:
```python
internal_ip_address = ays_host.model["nics"][0]["ipAddress"]
```

Get the address the public address and the VDC - will be user later:
```python
public_ip_address = vdc.model["externalnetworkip"]
```

Check the portforwards that got created for the host:
```python
vdc.portforwards
```

<a id="add-key"></a>
## Authorize your pubic SSH key

Get an ItsYou.online client object for your profile:
```python
iyo_user = j.clients.itsyouonline.get_user(app_id, secret)
```

Get the public key that is labelled as "default" from ItsYou.online:
```python
pubkey = iyo_user.public_keys.get(pubkey_label)
```

Or in case you didn't yet add your public key to your ItsYou.online profile, you may want to add it first:
```python
iyo_user.public_keys.add(pubkey_label, "<your public key string>")
```

Adding my "default" public key to authorized_keys for both cloudscalers and root:
```python
ays_host.prefab.system.ssh.authorize("cloudscalers", pubkey.model["publickey"])
ays_host.prefab.system.ssh.authorize("root", pubkey.model["publickey"])
```

<a id="install-jumpscale"></a>
##  Install JumpScale

Install JumpScale 9.3.0 (master):
```python
ays_host.prefab.js9.js9core.install(branch=branch)
```

<a id="install-ays"></a>
##  Install AYS Server

In order to install:
```python
ays_host.prefab.js9.atyourservice.install(branch=branch)
```

<a id="start-ays"></a>
##  Start and test the AYS server

Start the AYS server:
```python
ays_host.prefab.js9.atyourservice.start(host="0.0.0.0")
```

Add a port forward for the AYS server:
```python
ays_host.portforward_create(5000, 5000)
```

Test AYS Server:
```python
public_ays_url = "http://{}:{}".format(public_ip_address, "5000")
ays = j.clients.ays.get(public_ays_url)
ays.repositories.list()
```

<a id="api-console"></a>
## Add the AYS API console

Configure the AYS API Console:
```python
ays_host.prefab.js9.atyourservice.configure_api_console(url=public_ays_url)
```

This will update the value of `baseUri` in `JumpScale9AYS/ays/server/apidocs/api.raml`; make sure to use the public IP address here.

Test the AYS API Console in your browser by visiting:
http://<public_ip_address>:5000/apidocs/index.html?raml=api.raml

#http://195.143.34.145:5000/apidocs/index.html?raml=api.raml
http://195.143.34.146:5000/apidocs/index.html?raml=api.raml


<a id="add-portal"></a>
## Add JumpScale Portal

Optionally - first provision a new VM to host the portal:
```python
portal_host = vdc.machine_create(portal_host_name, "mykey", sizeId=3)
portal_host.prefab.system.ssh.authorize("cloudscalers", pubkey.model["publickey"])
portal_host.prefab.system.ssh.authorize("root", pubkey.model["publickey"])
portal_host.prefab.js9.js9core.install(branch=branch)
portal_host.prefab.tools.git.pullRepo("https://github.com/Jumpscale/ays9.git", branch=branch)
```

In the last command above we also clone the ays9 repository because of some file dependencies.

Or use the same host:
```python
portal_host = ays_host  
```

Or use an existing portal host:
```python
portal_host = vdc.machine_get(portal_host_name)
```

Execute the following to install JumpScale portal:
```python
portal_host.prefab.web.portal.install(False)
```

This also starts MongoDB (in the same tmux session as AYS server).

Start AYS Portal:
```python
portal_host.prefab.web.portal.start()
```

Add a port forward for the portal:
```python
portal_host.portforward_create(80, 8200)
```


<a id="add-cockpit"></a>
## Add the AYS Portal App

Add the AYS Portal "app":
```python
portal_host.prefab.js9.atyourservice.load_ays_space()
```

Stop and start the portal - still needed?:
```python
portal_host.prefab.web.portal.stop()
portal_host.prefab.web.portal.start()
```


<a id="configure-cockpit"></a>
## Configure the AYS Portal App

Configure the AYS Portal app:
```python
internal_ays_url = "http://{}:5000".format(internal_ip_address)
api_console_url = "http://{}:5000".format(public_ip_address)
portal_host.prefab.js9.atyourservice.configure_portal(ays_url=internal_ays_url, ays_console_url=api_console_url, portal_name="main")
```

The above will:
- Create and the value of `ays_uri` key in the `[portal.{portal_name}]` section of `jumpscale9.toml`
- Update the navigation link the portal (`nav.wiki`) for the AYS API console
- Automatically restart the portal


<a id="add-iyo"></a>
##  Add ItsYou.online integration

If not already done so, make sure the "ays-server-clients-org" organization was created, with an API access key:
```python
ays_clients_org = iyo_user.organizations.create(ays_clients_org_name)
ays_clients_org_api_key = ays_clients_org.api_keys.add(api_key_label, client_credentials_grant_type=True)
```

Two levels of ItsYou.online integration:
- [Integrate ItsYou.online into AYS server](#iyo-server)
- [Integrate ItsYou.online into AYS portal](#iyo-portal)

<a id="iyo-server"></a>
### Integrate ItsYou.online into AYS server

Configure the AYS server to only accept http/https requests with a JWT created for the organization created above:
```python
ays_host.prefab.js9.atyourservice.configure(production=True, organization=ays_clients_org_name)
```

This will:
- Create the `[ays]` section in `jumpscale9.toml` and create a key `production` which is set to true 
- Create the `[ays.oauth]` section under which the values of the keys `jwt_key` (currently hardcoded; so staging is not supported; or you need to change it manually) and `organization` are set

Here's is the code AYS uses to check the JWT:
https://github.com/Jumpscale/ays9/blob/master/JumpScale9AYS/ays/server/oauth2_itsyouonline.py#L35

It basically:
- Checks if the JWT was created for the `organization`
- If not, it checks if it was created for a user with membership to the `organization`
- If not, it checks if the `azp` in the JWT is equal to the `organization` - which simply comes down to trusting the portal, who can decide itself to which organization the user needs to be member 


We'll probably need to update the above implementation, one of the reasons us that we currently don't check the `aud` field in the JWT, which is against the recommendation about this in https://tools.ietf.org/html/rfc7519#section-4.1.3.

Optionally verify that the configuration was updated:
```bash
ssh root@<public_address> -p 2200
vim js9host/cfg/jumpscale9.toml
```

This is how it should look like:
```bash
[ays]
production = true

[ays.oauth]
organization = "ays-server-clients-org"
jwt_key = "MHYwEAYHKoZIzj0CAQYFK4EEACIDYgAES5X8XrfKdx9gYayFITc89wad4usrk0n27MjiGYvqalizeSWTHEpnd7oea9IQ8T5oJjMVH5cc0H5tFSKilFFeh//wngxIyny66+Vq5t5B0V0Ehy01+2ceEon2Y0XDkIKv"
```

Restart the AYS Server:
```python
ays_host.prefab.js9.atyourservice.stop()
ays_host.prefab.js9.atyourservice.start(host="0.0.0.0")
```

In order to test is, first get the API access key secret:
```python
ays_clients_org = iyo_user.organizations.get(ays_clients_org_name)
ays_clients_org_api_key = ays_clients_org.api_keys.get(api_key_label)
ays_clients_org_secret = ays_clients_org_api_key.model["secret"]
```

Re-test AYS Server with ItsYou.online integration active:
```python
ays = j.clients.ays.get(public_ays_url, ays_clients_org_name, ays_clients_org_secret)
ays.repositories.list()
```

In order to also test the AYS API Console with ItsYou.online integration, let's first get a JWT, using the IYO client:
```python
jwt = j.clients.itsyouonline.get_jwt(ays_clients_org_name, ays_clients_org_secret)
```

OR - using the AYS client:
```python
jwt2 = ays._getJWT(ays_clients_org_name, ays_clients_org_secret)
```

OR - you can also create the JWT with the AYS command line tool:
- `ays generatetoken --clientid $GLOBAL_ID --clientsecret $KEY_SECRET --validity 3600`
- `ays generatetoken --clientid $APP_ID --clientsecret $SECRET --organization $ITSYOUONLINEORG --validity 3600`

Both work, in the second case the JWT is for a user that needs to be member of `ays-server-clients-org` 

http://<public_ip_address>:5000/apidocs/index.html?raml=api.raml
#http://195.143.34.203:5000/apidocs/index.html?raml=api.raml

Authentication: Bearer $jwt


<a id="iyo-portal"></a>
### Integrate ItsYou.online into AYS portal

If not already done, create the ItsYou.online organization where the portal users need to be member:
```python
ays_portal_users_org = iyo_user.organizations.create(portal_users_org_name)
```

Prepare the callback URL:
```python
redirect_url = "http://{}:{}".format(public_ip_address, 8200)
```

Configure `client_id` and `client_secret` that the portal uses the identify itself, and also configure the ItsYou.online `organization` of which portal users need to be member:
```python
portal_host.prefab.web.portal.configure(mongodbip='127.0.0.1', mongoport=27017, production=True, client_id=ays_clients_org_name, client_secret=ays_clients_org_secret, scope_organization=portal_users_org_name, redirect_address=redirect_url)
```

> Note: The values set for `client_id` and `client_secret` are only used by the portal to authenticate itself when authenticating portal users (authorization code grant flow), when the portal consumes the AYS server RESTful API it forwards JWT from the user; so the portal doesn't create a JWT based on the client_id and secret

> Also note that `client_id` and `client_secret` are ignored when production = FALSE

Check the configuration on the host:
```bash
vim js9host/cfg/jumpscale9.toml
```

Stop and start the portal:
```python
portal_host.prefab.web.portal.stop()
portal_host.prefab.web.portal.start()
```

Also update redirect url in ItsYou.online:
```python
ays_clients_org_api_key.update(callback_url=redirect_url)
```

<a id="caddy"></a>
## Add Caddy

Optionally - delete the Caddy host:
```python
caddy_host = vdc.machine_get(caddy_host_name)
caddy_host.delete()
```

Create the Caddy host:
```python
caddy_host = vdc.machine_create(caddy_host_name, "mykey", sizeId=3)
caddy_host.prefab.system.ssh.authorize("cloudscalers", pubkey.model["publickey"])
caddy_host.prefab.system.ssh.authorize("root", pubkey.model["publickey"])
```

Install Caddy:
```python
caddy_host.prefab.web.caddy.install()
```

```python
caddy_domain = "ays.vreegoebezig.be"
tls_email = "yves@vreegoebezig.be"
portal_internal_ip_address = portal_host.model["nics"][0]["ipAddress"]
caddy_cfg = """#tcpport:80

http://{0} {{
  redir https://{0}
}}

https://{0} {{
  proxy /api {1}:5000 {{
      without /api
      transparent
  }}

  proxy /apidocs {1}:5000 {{
      transparent
  }}
  
  proxy / {2}:8200 {{
    transparent
  }}
  
  gzip
  log
  tls {3}
}}
""".format(caddy_domain, internal_ip_address, portal_internal_ip_address, tls_email)
```

Port forward
```python
caddy_host.portforward_create(443, 443)
caddy_host.portforward_create(80, 80)
```

```python
caddy_cfg_path = "/opt/cfg/caddy.cfg"
caddy_host.prefab.core.file_write(location=caddy_cfg_path, content=caddy_cfg)
```

Update the redirect url, both for the portal and at ItsYou.online:
```python
redirect_url = "https://{}".format(caddy_domain)
portal_host.prefab.web.portal.configure(mongodbip='127.0.0.1', mongoport=27017, production=True, client_id=ays_clients_org_name, client_secret=ays_clients_org_secret, scope_organization=portal_users_org_name, redirect_address=redirect_url)
ays_clients_org_api_key.update(callback_url=redirect_url)
```

Update the AYS API Console (`api.raml`):
```python
public_ays_url = "https://{}/api".format(caddy_domain)
ays_host.prefab.js9.atyourservice.configure_api_console(url=public_ays_url)
```

Update the AYS Portal app configuration:
```python
api_console_url = "https://{}/api".format(caddy_domain)
portal_host.prefab.js9.atyourservice.configure_portal(ays_url=internal_ays_url, ays_console_url="https://{}".format(caddy_domain), portal_name="main")
```

NOT NEEDED IF EXECUTED PREVIOUS STEP - Restart the portal:
```python
portal_host.prefab.web.portal.stop()
portal_host.prefab.web.portal.start()
```

Start Caddy:
```python
#caddy_host.prefab.web.caddy.stop()
caddy_host.prefab.web.caddy.start()
```

Test AYS:
```python
#public_ays_url = "http://{}:5000".format(public_ip_address)
public_ays_url = "https://{}/api".format(caddy_domain)
ays = j.clients.ays.get(public_ays_url, ays_clients_org_name, ays_clients_org_secret)
ays.repositories.list()
```