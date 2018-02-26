# Changes in 9.3.0

Host and port are configured now through the `configure()` method:
```python
j.tools.prefab.local.js9.atyourservice.configure(production=False, organization='', restart=True, host=None, port=None)
```

Example:
```python
j.tools.prefab.local.js9.atyourservice.configure(host="0.0.0.0", port=5000)
```

For each repository you need to configure the ItsYou.online client, using the RESTful API:
```yaml
{
	"instance": "main",
	"jslocation": "j.clients.itsyouonline",
	"data": {
		"baseurl": "https://itsyou.online/api",
		"application_id_": "{{ app_id  }}",
		"secret_": "{{ secret  }}"
	}
}
```

Or from JumpScale:
```python
ays_url = "http://localhost:5000"
repo_name = "insomnia"
ays = j.clients.atyourservice.get(base_uri=ays_url)
repo = ays.api.ays.getRepository(repository=repo_name)


@todo #figure out how to do it using JumpScale; I've only been able to achieve it using Insomnia
# ays.api.ays.configure_jslocation()
# locally you;d use this:
# j.tools.configmanager.configure(location='', instance='main', data={}, sshkey_path=None)
```

Do the same for OpenvCloud:
```yaml
{
	"instance": "main",
	"jslocation": "j.clients.openvcloud",
	"data": {
		"address": "{{ ovc_url  }}"
	}
}
```

As a result a new configuration instance `insomnia_main` (`$repositoryname`_`$instancename`) will have been added, check locally:
```python
j.tools.configmanager.list(location="j.clients.openvcloud")
```

Also see:
https://github.com/openvcloud/ays_templates/tree/master/docs/OVC_Client

List all templates:
```python
resp = ays.api.ays.listAYSTemplates()
for item in resp.json():
    print(item["name"])
```

Add templates:
```python
ovc_templates_url = "https://github.com/openvcloud/ays_templates"
ovc_templates_branch = "master"
data = {
    'url': ovc_templates_url,
    'branch': ovc_templates_branch
}
resp = ays.api.ays.addTemplateRepo(data=data)
```

New blueprint:
```yaml
g8client__cl:
    instance: "main"
    account: "Account_of_Yves"

vdc__myvdc:
    g8client: "cl"
    location: "ch-gen-1"

actions:
    - action: "install"
```

AYS repositories are now in `/opt/var/ays_repos`.