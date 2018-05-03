# JumpScale Sandbox


Source:
https://github.com/Jumpscale/sandbox/blob/master/README.md#an-example-to-run-sandboxing-on-the-local-machine
https://github.com/Jumpscale/sandbox/blob/master/README.md#testing-the-sandbox


## Creating your Sandbox flist

First you need to make sure that you have the Zero-Hub client installed, and up to date:
```bash
pip install -e 'git+https://github.com/zero-os/0-hub#egg=zerohub&subdirectory=client' --upgrade
js9_init
```

Then you need to make sure that the Zero-Hub client is configured correctly:
```python
iyo_username = 'yves'
iyo_client = j.clients.itsyouonline.get('main')
zhub_cfg = {'token_': iyo_client.jwt, 'username': iyo_username,'url': 'https://hub.gig.tech/api'}
zhub_client = j.clients.zerohub.get(data=zhub_cfg)
zhub_client.config.save()
```

This will run the sandboxing of js9 on the local machine without the need of a remote machine.

Clone the [JumpScale sandbox](https://github.com/Jumpscale/sandbox) repository:
```bash
cd /opt/code/github/jumpscale/
git clone https://github.com/Jumpscale/sandbox.git
```

Then run the `sandbox_js9_local.py` script:
```python
python3 /opt/code/github/jumpscale/sandbox/sandbox_js9_local.py
```

The script will do the following:
- Create the js9 sandbox flist and upload it to the current user's (configured via IYO) account at https://hub.gig.tech/
- Merge the uploaded flist with a base Ubuntu 16.04 flist (gig-official-apps/ubuntu1604-for-js.flist)
- Upload the merged flist as <username>/js9_sandbox_full.flist


## Using your sandbox flist

To test the uploaded flist you can use the `zbundle` tool from https://github.com/zero-os/0-bundle 

After creating the sandbox, a flist for js9 will be uploaded to your account on your account at https://hub.gig.tech

To install the `zbundle` tool you need to execute the following steps:
```bash
mkdir -p /opt/bin
wget -O /opt/bin/zbundle https://download.gig.tech/zbundle
cd /opt/bin
chmod +x zbundle
```

Then you can start your sandbox using the following command:
```bash
your_zero_hub_account="abdelrahman_hussein_1"
flist_url=https://hub.gig.tech/$your_zero_hub_account/js9_sandbox_full.flist
zbundle -id js9sandbox --entry-point /bin/bash --no-exit $flist_url
```

Once started you can the JumpScale interactive shell from the sandboxed environment:
```bash
cd /tmp/zbundle/js9sandbox
echo "nameserver 8.8.8.8" > /etc/resolv.conf
source /env.sh; js9
```

The second command above is for making sure you can connect to the internet.