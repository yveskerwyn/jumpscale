# Bootstrap

The below steps document how to install JumpScale 9.2.1 on a virtual machine in an OpenvCloud environment.

Once you have this machine you have two options:
- [Install AYS Server locally on this machine](local_ays_setup.md)
- [Install AYS Server on a remote machine, residing in another cloud space](remote_ays_setup.md)

Steps:
- [Prepare](#prep)
- [Authorize your SSH key](#authorize)
- [Install the Z Utilities](#install-bash-tools)
- [Install JumpScale](#install-jumpscale)
- [Install JOSE Python package](#jose)
- [Save your ItsYou.online API key](#api-key)
- [Save your GitHub PAT](#github-pat)
- [Test OVC Client](#test-ovc)

<a id="prep"></a>
## Prepare

- Get a personal access token on GitHub
- Create your ItsYou.online user profile
- Save your SSH public key to your ItsYou.online profile, label it as "default" 
- Create an API key (application ID + secret) on your ItsYou.online profile
- Request access to an OpenvCloud account by contacting an OpenvCloud account administrator
- Logon to the OpenvCloud environment using your ItsYou.online credentials
- Create a "bootstrap" cloud space using the OpenvCloud account
- Create a "bootstrap" virtual machine in this "bootstrap" cloud space
- Create a port forward for port 22 on the "bootstrap" VM

<a id="authorize"></a>
## Authorize your SSH key

Login to the "bootstrap" VM:
```bash
ssh cloudscalers@195.134.212.37 -p7122
```

Create `.ssh` directory:
```bash
sudo mkdir ~/.ssh
```

Add your SSH public key to a new `authorized_keys` file:
```bash
sudo vim ~/.ssh/authorized_keys
```

Do the same for the root user:
```bash
sudo -i
vim ~/.ssh/authorized_keys
```

Log out, twice:
```bash
exit
exit
```

<a id="install-bash-tools"></a>
## Install the Z Utilities

Login again with root:
```bash
ssh root@195.134.212.37 -p7122
```

Install the Z tools
```bash
export ZUTILSBRANCH=9.2.1
curl https://raw.githubusercontent.com/Jumpscale/bash/${ZUTILSBRANCH}/install.sh?$RANDOM > /tmp/install.sh
bash /tmp/install.sh
```

Source the newly created `bash_profile` file:
```bash
. ~/.bash_profile
```

<a id="install-jumpscale"></a>
## Install JumpScale

```bash
export JS9BRANCH=9.2.1
ZInstall_host_js9_full
```

<a id="jose"></a>
## Install JOSE Python package

Execute:
```bash
pip install python-jose
```

<a id="api-key"></a>
## Save your ItsYou.online API key

Save your API key to environment variables:
```bash
vim ~/.bash_profile
```

Add the application ID and secret:
```bash
export APP_ID="..."
export SECRET="..."
```

Make sure to source the updated `bash_profile`:
```bash
. ~/.bash_profile
```

<a id="github-pat"></a>
## Save your GitHub PAT

Do the same for your personal access token from GitHub:
```bash
export GITHUB_PAT="..."
```

Again make sure to source the updated bash_profile.


<a id="test-ovc"></a>
## Test OVC Client

Start the interactive shell:
```bash
js9
````

List all OVC accounts you have access to:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
ovc_url = "be-gen-1.demo.greenitglobe.com"
ovc_client = j.clients.openvcloud.get(applicationId=app_id, secret=secret, url=ovc_url)
ovc_client.accounts
```