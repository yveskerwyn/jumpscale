# Systemd Unit Files

## AYS server
```bash
vim /etc/systemd/system/ays.service
[Unit]
Description=AYS Server
Wants=network-online.target
After=network-online.target

[Service]
WorkingDirectory=/opt/code/github/jumpscale/ays9
ExecStart=/usr/bin/sudo /bin/bash -a -c 'source /root/.bash_profile && /usr/bin/env /usr/bin/python3 main.py --host 0.0.0.0 --port 5000 --log info'

[Install]
WantedBy=multi-user.target
```

If you ever change this file, you need to reload the services:
```bash
systemctl daemon-reload
```

After having created the above unit file for the AYS server, have systemd enable it:
```bash
systemctl enable ays
```

List all enabled services:
```bash
systemctl list-unit-files | grep enabled
```

Start the AYS service:
```bash
systemctl start ays.service
```

List all running services:
```bash
systemctl | grep running
```

In order to check the output from AYS server you might start a TMUX session and tail the log file:
```bash
tmux new -s "atyourservice"
tail -f /opt/var/log/jumpscale.log
```

## AYS Portal

@TODO: also create a unit file for mongodb
@ root@portal:~# echo **START**;/opt/bin/mongod --dbpath '/opt/var/data/mongodb' && echo **OK** || echo **ERROR**

```bash
vim /etc/systemd/system/portal.service
[Unit]
Description=AYS Portal
Wants=network-online.target
After=network-online.target

[Service]
WorkingDirectory=/opt/jumpscale9/apps/portals/main
ExecStart=/usr/bin/sudo /bin/bash -a -c 'source /root/.bash_profile && /usr/bin/env /usr/bin/python3  portal_start.py --instance main'

[Install]
WantedBy=multi-user.target
```


If you ever change this file, you need to reload the services:
```bash
systemctl daemon-reload
```

After having created the above unit file for the AYS portal, have systemd enable it:
```bash
systemctl enable portal
```

List all enabled services:
```bash
systemctl list-unit-files | grep enabled
```

Start the portal service:
```bash
systemctl start portal.service
```

List all running services:
```bash
systemctl | grep running
```


## Caddy

```bash
vim /etc/systemd/system/caddy.service
[Unit]
Description=At Your Service
Wants=network-online.target
After=network-online.target

[Service]
WorkingDirectory=/
ExecStart=/bin/bash -a -c 'ulimit -n 8192;/opt/bin/caddy -conf=/opt/cfg/caddy.cfg  -agree'

[Install]
WantedBy=multi-user.target
```

After having created the above files, have systemd enable them:
```bash
sudo systemctl enable ays portal caddy
```

List all enabled services:
```bash
systemctl list-unit-files | grep enabled
```

Start the services:
```bash
systemctl start ays.service
systemctl start portal.service
systemctl start caddy.service
```

List all running services:
```bash
systemctl | grep running
```
