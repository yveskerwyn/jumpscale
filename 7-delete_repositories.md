# Delete Repositories

Optionally delete the AYS repositories that are created as part of a default AYS installation.

First connect to your AYS server:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]

caddy_domain = "ays.portal.gigeurope.tech"
public_ays_url = "https://{}/api".format(caddy_domain)
ays_clients_org_name = "ays-organizations.ch-gen-1-artilium"
scope = 'user:memberof:{}'.format(ays_clients_org_name)
jwt = j.clients.itsyouonline.get_jwt(client_id=app_id, secret=secret, scope=scope)
ays = j.clients.ays.get_with_jwt(url=public_ays_url, jwt=jwt)
```

The go through the list of all known AYS repositories and delete them:
```python
for repo_name in ["sample_repo_basicdelete", "sample_repo_scheduler",  "sample_repo_scheduler", "sample_repo1", "sample_repo2", "sample_repo3", "sample_repo4", "sample_repo5","sample_repo_delete_models", "sample_repo_timeout", "sample_repo_recurring", "sample_repo_longjobs", "sample_repo_account", "kvm_packet.net", "sample_repo_consume", "docker_ovc"]:
    try:
        repo = ays.repositories.get(name=repo_name)
        repo.delete()
    except:
        pass
```