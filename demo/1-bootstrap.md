# Bootstrap

Steps:
- [Prepare](#prep)
- [Authorize your SSH key](#authorize)
- [Install the Z Utilities](#install-bash-tools)
- [Install JumpScale](#install-jumpscale)
- [Install JOSE Python package](#jose)
- [Get your API key](#api-key)
- [Test OVC Client](#test-ovc)

<a id="prep"></a>
## Prepare

- Create your ItsYou.online user profile
- Logon to an OpenvCloud environment using your ItsYou.online credentials
- Contact an OpenvCloud adminstrator to grant access to an OpenvCloud account
- Create a "bootstrap" cloud space on be-gen-1
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
curl https://raw.githubusercontent.com/Jumpscale/bash/${ZUTILSBRANCH}/install.sh?$RANDOM > /tmp/install.sh;bash
/tmp/install.sh
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
## Get your API key

Save your API key to environment variables:
```bash
vim ~/.bash_profile
```

Add the application ID and secret:
```bash
export APP_ID="..."
export SECRET="..."
```

Source the updated `bash_profile`:
```bash
. ~/.bash_profile
```

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
ovc_client.accounts
```