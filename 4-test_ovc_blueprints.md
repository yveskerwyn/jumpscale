# Test OpenvCloud Blueprints

Make sure you have a working connection to your AYS Server, see [Test AYS Connectivity](3-test_ays_connectivity.md) for details.

Variables:
```python
import yaml
ovc_templates_branch = "9.2.1"
test_repo_name="4sergey"
ovc_url = "be-gen-1.demo.greenitglobe.com
test_account_name = "Account_of_Yves"
location = "be-gen-1"
test_vdc_name = "test-vdc"
test_vm_name = "test-vm"
```

First make sure you have added the OpenvCloud templates add to your AYS server:
```python
ays.templates.addTemplates(repo_url="https://github.com/openvcloud/ays_templates", branch=ovc_templates_branch)
```

> You will have to type `yes` in the tmux session of AYS.

(Optionally) create a GitHub repository using your GitHub personal access token:
```python
token = os.environ["GITHUB_PAT"]
github = j.clients.github.get(token)
github_repo = github.repo_create(test_repo_name, private=False)
#github_repo = github.getRepo(test_repo_name)
repo_ssh_url = github_repo.ssh_url
```

(Optionally) add the SSH key used by the AYS server to GitHub - only required for private repositories:
```python
#ays_pub_key = j.clients.ssh.SSHKeyGetFromAgentPub('/root/.ssh/ays_repos_key')
ays_pub_key = ays_host.prefab.core.file_read('/root/.ssh/ays_repos_key.pub')
user = github.api.get_user()
user.create_key("ays", ays_pub_key)
```

Create a repository:
```python
ays_repo = ays.repositories.create(test_repo_name, repo_ssh_url)
```

Get a JWT to pass in the blueprint in order to access the OpenvCloud Cloud API:
```python
jwt = j.clients.itsyouonline.get_jwt(app_id, secret)
```

Prepare a blueprint for each AYS service definition:
```python
bp_g8client = dict()
bp_g8client['g8client__cl'] = {'url': ovc_url, 'jwt': jwt, 'account': test_account_name}

print(yaml.dump(bp_g8client, default_flow_style=False))

bp_users = dict()
bp_users['uservdc__lee'] = {'g8client': 'cl', 'provider': 'itsyouonline', 'email': 'lee@gig.tech'}
#bp_users['uservdc__greyse'] = 'g8client': 'cl', 'provider': 'itsyouonline', 'email': '...'}

users = [{'name': 'lee', 'accesstype': 'ARCXDU'}]

bp_vdc = dict()
bp_vdc['vdc__{}'.format(test_vdc_name)] = {'g8client': 'cl', 'location': location, 'uservdc': users}

print(yaml.dump(bp_vdc, default_flow_style=False))

bp_vm = dict()
bp_vm['node.ovc__{}'.format(test_vm_name)] = {'vdc': test_vdc_name, 'bootdisk.size': 20, 'sizeID': 1, 'os.image': 'Ubuntu 16.04 x64'}

print(yaml.dump(bp_vm, default_flow_style=False))
```

Create a method to execute the above blueprints:
```python
def execute_bp(repo, bp_name, blueprint):
    import yaml
    serialized_blueprint = yaml.dump(blueprint, default_flow_style=False)
    bp = repo.blueprints.create(bp_name, serialized_blueprint)
    bp.execute()
```

Execute the blueprints:
```python
execute_bp(ays_repo, "client.yaml", bp_g8client)
execute_bp(ays_repo, "users.yaml", bp_users)
execute_bp(ays_repo, "vdc.yaml", bp_vdc)
execute_bp(ays_repo, "vm.yaml", bp_vm)
```

Create one more blueprint for scheduling execution on the install actions:
```python
bp_actions = dict()
bp_actions['actions'] = [{'action': 'install'}]
```

Execute the actions blueprint:
```python
execute_bp(ays_repo, "actions.yaml", bp_actions)
```

Create and start a run:
```python
run_id = ays_repo.runs.create(execute=True)
```

Let's add disks:
```python
bp_disks = dict()
bp_disks['disk.ovc__db01'] = {'type': 'D', 'size': 150, 'maxIOPS': 300}
bp_disks['disk.ovc__db02'] = {'type': 'D', 'size': 150, 'maxIOPS': 300}
bp_disks['disk.ovc__db03'] = {'type': 'D', 'size': 150, 'maxIOPS': 300}

execute_bp(ays_repo, "disks.yaml", bp_disks)

actions = ays_repo.blueprints.get("actions.yaml")
actions.execute()

disks = ['db01', 'db02', 'db03']

bp_add_disks = dict()
bp_add_disks['node.ovc__{}'.format(test_vm_name)]  = {'disk': disks}

print(yaml.dump(bp_add_disks, default_flow_style=False))

execute_bp(ays_repo, "add_disks.yaml", bp_add_disks)

actions.execute()

run_id = ays_repo.runs.create(execute=True)
```