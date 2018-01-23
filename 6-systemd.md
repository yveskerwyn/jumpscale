# Systemd Unit Files

## AYS server
```bash
cat /etc/systemd/system/ays.service
[Unit]
Description=AYS Server
Wants=network-online.target
After=network-online.target

[Service]
WorkingDirectory=/opt/code/github/jumpscale/ays9
ExecStart=/usr/bin/python3 main.py --host 127.0.0.1 --port 5000 --log info

[Install]
WantedBy=multi-user.target
```

## AYS Portal
```bash
cat /etc/systemd/system/portal.service
[Unit]
Description=AYS Portal
Wants=network-online.target
After=network-online.target

[Service]
WorkingDirectory=/opt/jumpscale9/apps/portals/main 
ExecStart=/usr/bin/python3 portal_start.py --instance main

[Install]
WantedBy=multi-user.target
```

## Caddy

```bash
cat /etc/systemd/system/caddy.service
[Unit]
Description=At Your Service
Wants=network-online.target
After=network-online.target

[Service]
WorkingDirectory=/opt/bin/caddy
ExecStart=ulimit -n 8192; /opt/bin/caddy -conf=/opt/cfg/caddy.cfg  -agree

[Install]
WantedBy=multi-user.target
```

After having created the above files, have systemd reload its configuration:
```bash
sudo systemctl daemon-reload
```

Enable the services:
```bash
sudo systemctl enable ays portal caddy
```