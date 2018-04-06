# 0-robot Client

Make sure:
- Your configuration manager is initialized
- The SSH key used by the config manager is associated with your Git server account

Then:
- [Get a JWT](#get-jwt)
- [Start the 0-robot](#start-robot)
- [Create configuration instance for the 0-robot](#zrbot-config)
- [Create the 0-robot services](#create-services)

<a id="get-jwt"></a>
## Get a JWT

In order to get a JWT we need an application ID Ans secret from ItsYou.online.

If not already done so before, create an ItsYou.online configuration instance, here from an application ID and secret available from environment variable:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
iyo_config = {
    "application_id_": app_id,
    "secret_": secret
}

iyo_client = j.clients.itsyouonline.get(instance='main', data=iyo_config, create=True, die=True, interactive=False)
```

This configuration instance only needs to be created once, getting it is simple:
```python
iyo_client = j.clients.itsyouonline.get(instance='main')
```

Using the configuration instance for ItsYou.online you can easily get a (refresheable) JWT:
```python
jwt = iyo_client.jwt_get(refreshable=True)
```

<a id="start-robot"></a>
## Have 0-robot running

Prepare by:
- Making sure you have a Zero-Robot `data` repository and a JumpScale `configuration` repository available
- Loading a SSH key named `id_rsa`

Clone the 0-robot repository:
```bash
mkdir -p /opt/code/github/zero-os
cd /opt/code/github/zero-os
git clone https://github.com/zero-os/0-robot.git
```

Install Zero-Robot and all dependencies:
```bash
cd /opt/code/github/zero-os/0-robot
export ZROBOTBRANCH="master"
git checkout $ZROBOTBRANCH
pip install -r requirements.txt
pip install .
```

Create two private Git repositories:
- One for your JumpScale configuration instances
- Another one for the 0-robot data

Set environment variables:
```bash
config_repo="ssh://git@docs.greenitglobe.com:10022/yves/jsconfig-robot.git"
data_repo="ssh://git@docs.greenitglobe.com:10022/yves/robotdata.git"
template_repo_openvcloud="https://github.com/openvcloud/0-templates.git"
export internal_ip_address="$(ip route get 8.8.8.8 | awk '{print $NF; exit}')"
#port_forwarding="$internal_ip_address:8000:6600"
```

Start 0-robot locally - in the background:
```bash
zrobot server start -D $data_repo -C $config_repo -T $template_repo_openvcloud --auto-push --auto-push-interval 30 &
<ENTER>
```


<a id="zrbot-config"></a>
## Create configuration instance for the 0-robot

Check for an existing 0-robot config instance:
```python
j.clients.zrobot.list()
```

If there is an existing one, get it and check the configuration, for the instance `main`:
```python
zrcl = j.clients.zrobot.get(instance="main")
zrcl.config
```

Example output:
```python
config:j.clients.zrobot:main


    jwt_ = ''

    url = 'http://192.168.103.251:6600'
```

If not done so before, create a new configuration instance for your 0-robot:
```python
import os
internal_ip_address = os.environ["internal_ip_address"]
zero_robot_port = 6600
zr_config = {}
#zr_config["jwt_"] = ''
zr_config["url"] = '{}:{}'.format(internal_ip_address, zero_robot_port)
zrcl = j.clients.zrobot.get(instance="main", data=zr_config, create=True, interactive=False)
```

In order to update the configuration:
```python
zr_cfg = zrcl.config
zr_cfg.data_set(key="url", val="http://localhost:6600", save=True)
```


<a id="create-services"></a>
## Create the 0-robot services

List 0-robot services:
```python
zrcl.api.services.listServices()
```

Or better use the wrapper client:
```python
robot = j.clients.zrobot.robots['main']
robot.services.names
```

Get one of them, if any:
```python
service = robot.services.get(name="ch-gen-1")
```

List all actions available for this service:
```python
service.actions
```

Delete the service:
```python
s.delete()
```

Variables for using the OpenvCloud templates:
```python
OVC_TEMPLATE_UID = "github.com/openvcloud/0-templates/openvcloud/0.0.1"
ACCOUNT_TEMPLATE_UID = "github.com/openvcloud/0-templates/account/0.0.1"
VDC_TEMPLATE_UID = "github.com/openvcloud/0-templates/vdc/0.0.1"
SSHKEY_TEMPLATE_UID = "github.com/openvcloud/0-templates/sshkey/0.0.1"
VM_TEMPLATE_UID = "github.com/openvcloud/0-templates/node/0.0.1"

ovc_location = "be-gen-1"
ovc_address = "be-gen-1.demo.greenitglobe.com"
ovc_jwt = iyo_client.jwt_get(refreshable=True)

ovc_service_name = "be-gen-1"
ovc_account_name = "Account_of_Yves"
vdc_name = "test-vdc"
```

Create an OpenvCloud client service:
```python
ovc_client_data = {}
ovc_client_data["location"] = ovc_location
ovc_client_data["address"] = ovc_address
ovc_client_data["token"] = ovc_jwt

ovc_service = robot.services.create(template_uid=OVC_TEMPLATE_UID, service_name=ovc_service_name, data=ovc_client_data)
```

Create an OpenvCloud account service:
```python
ovc_account_data = {}
ovc_account_data["description"] = "test"
ovc_account_data["openvcloud"] = ovc_service_name
ovc_account_data["location"] = ovc_location
account_service = robot.services.create(template_uid=ACCOUNT_TEMPLATE_UID, service_name=ovc_account_name, data=ovc_account_data)
```

Create a VDC service:
```python
ovc_account_service_name = "Account_of_Yves"
vdc_name = "test-vdc"
ovc_location = ovc_location

vdc_data = {}
vdc_data["account"] = ovc_account_service_name
vdc_data["location"] = ovc_location
#vdc_data["VdcUser"] = ...
vdc_service = robot.services.create(template_uid=VDC_TEMPLATE_UID, service_name=vdc_name, data=vdc_data)
```

Install the VDC:
```python
ovc_service.schedule_action(action="install")
account_service.schedule_action(action="install")
vdc_service.schedule_action(action="install")
```

Before deploying a virtual machine (node), we first create service for a SSH key. This SSH key will be needed for creating the virtual machine:
```python
sshkey_name = "mykey"
passphrase = "atleast5chars"
#sshkey_path = "/root/.ssh/"
sshkey_data = {}
#sshkey_data["dir"] = sshkey_path 
sshkey_data["passphrase"] = passphrase
sshkey_service = robot.services.create(template_uid=SSHKEY_TEMPLATE_UID, service_name=sshkey_name, data=sshkey_data)
```

Create VM service:
```python
vm_name = "my-vm"
vm_data = {}
vm_data["sshKey"] = sshkey_name
vm_data["vdc"] = vdc_name
vm_service = robot.services.create(template_uid=VM_TEMPLATE_UID, service_name=vm_name, data=vm_data)
```

Install the VM:
```python
sshkey_service.schedule_action(action="install")
vm_service.schedule_action(action="install")
```


<a id="blueprints"></a>
## Using blueprints

Create a blueprint:
```python
bp_ovc = dict()
ovc_client_service_name = "ch-gen-1"
ovc_address = "ch-gen-1.gig.tech"
ovc_location = "ch-gen-1"
ovc_jwt = "eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJlMnpsTi03U0M2N3RhdjN0UlJuZG9VQUd4a1U1IiwiZXhwIjoxNTIyMjUwMjgyLCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieXZlcyJ9.i1LGebmiDc3B4B-LAQibKCff8BMDKH9DT1UTa7gsgIDRLLRGQv2lQFqwZsZwt57IsdEPQcl8gASCcXBaMH6qhJK_ggJGLNjo_SpVtrUlWQzVZwlFiKMLOQQQxLp6oSLV"
bp_ovc['github.com/openvcloud/0-templates/openvcloud/0.0.1__{}'.format(ovc_client_service_name)] = {'address': ovc_address, 'location': ovc_location, 'token': ovc_jwt}
import yaml; print(yaml.dump(bp_ovc, default_flow_style=False))


bp_account = dict()
ovc_account_service_name = "Account_of_Yves"
bp_account['github.com/openvcloud/0-templates/account/0.0.1__{}'.format(ovc_account_service_name)] = {'openvcloud': ovc_client_service_name}
import yaml; print(yaml.dump(bp_account, default_flow_style=False))

bp_vdcuser = dict()
iyo_username = "yves"
user_email_address = "yves@gig.tech"
bp_vdcuser['github.com/openvcloud/0-templates/vdcuser/0.0.1__{}'.format(iyo_username)] = {'openvcloud': ovc_client_service_name, 'provider': 'itsyouonline', 'email': user_email_address}
import yaml; print(yaml.dump(bp_vdcuser, default_flow_style=False))


bp_vdc = dict()
vdc_name = "my_vdc"
bp_vdc['github.com/openvcloud/0-templates/vdc/0.0.1__{}'.format(vdc_name)] = {'account': ovc_account_service_name,  'location': ovc_location, 'users': [{'name': iyo_username, 'accesstype': 'ARCXDU'}]}
import yaml; print(yaml.dump(bp_vdc, default_flow_style=False))


bp_services = dict()
bp_services['services'] = [bp_ovc, bp_account, bp_vdcuser, bp_vdc]
import yaml; print(yaml.dump(bp_services, default_flow_style=False))


bp_actions = dict()
bp_actions['actions'] = [{'actions': ['install']}]
import yaml; print(yaml.dump(bp_actions, default_flow_style=False))


bp_allinone = dict(bp_services)
bp_allinone.update(bp_actions) 
import yaml; print(yaml.dump(bp_allinone, default_flow_style=False))
```

Execute blueprint:
```python
#content = j.data.serializer.yaml.loads(bp_allinone)
serialized_blueprint = yaml.dump(bp_allinone, default_flow_style=False)

data = {'content': serialized_blueprint}

zrcl1.api.blueprints.ExecuteBlueprint(data)

try:
    tasks, _ = client.api.blueprints.ExecuteBlueprint(data)
    result = dict()
    for task in tasks:
        if task.service_name in result.keys():
            result[task.service_name].update({task.action_name: task.guid})
        else:
            result[task.service_name] = {task.action_name: task.guid}
    return result
except HTTPError as err:
    msg = err.response.json()['message']
    self.log('message: %s' % msg)
    self.log('code: %s' % err.response.json()['code'])
    return msg
```




Here is the blueprint:
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__be-gen-1:
        address: 'be-gen-1.demo.greenitglobe.com'
        location: 'be-gen-1'
        token: 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJlMnpsTi03U0M2N3RhdjN0UlJuZG9VQUd4a1U1IiwiZXhwIjoxNTIxNzk4ODE1LCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieXZlcyJ9.3_WkvHcCvMDCRCwYy3tnqUnNQRE0OUACKkH57xUqQ5TNgPYF0FVigxFDNcPjrgOXU3ARoz6z1UN2PMaeSxHMRFH2AL8BPxwLVUz0WaP1YfYLzx2My_nYO8Q7obS83sw3'

    - github.com/openvcloud/0-templates/account/0.0.1__gigdevelopers:
        openvcloud: be-gen-1

    - github.com/openvcloud/0-templates/vdcuser/0.0.1__yves:
        openvcloud: be-gen-1
        provider: itsyouonline
        email: yves@gig.tech
        
    - github.com/openvcloud/0-templates/vdc/0.0.1__kuberyves:
        account: gigdevelopers
        location: be-gen-1
        users:
            - name: yves
              accesstype: CXDRAU
    
actions:
    actions: ['install']
```






```python
from js9 import j
from jinja2 import Environment, FileSystemLoader

self.j2_env = Environment(loader=FileSystemLoader(searchpath=templatespath), trim_blocks=True)



def handle_blueprint(self, yaml, **kwargs):
    kwargs['token'] = self.iyo_jwt()
    blueprint = self.create_blueprint(yaml, **kwargs)
    return self.execute_blueprint(blueprint)


def create_blueprint(self, yaml, **kwargs):
    """
    yaml file that is used for blueprint creation
    """
    blueprint = self.j2_env.get_template('base.yaml').render(services=yaml,
                                                                actions='actions.yaml',
                                                                **kwargs)
    return blueprint


def execute_blueprint(self, blueprint):
    os.system('echo "{0}" >> /tmp/{1}_{2}.yaml'.format(blueprint, self._testID,
                                                        self.random_string()))
    instance, _ = utils.get_instance()
    client = j.clients.zrobot.get(instance)


    content = j.data.serializer.yaml.loads(blueprint)
    data = {'content': content}
    try:
        tasks, _ = client.api.blueprints.ExecuteBlueprint(data)
        result = dict()
        for task in tasks:
            if task.service_name in result.keys():
                result[task.service_name].update({task.action_name: task.guid})
            else:
                result[task.service_name] = {task.action_name: task.guid}
        return result
    except HTTPError as err:
        msg = err.response.json()['message']
        self.log('message: %s' % msg)
        self.log('code: %s' % err.response.json()['code'])
        return msg
```