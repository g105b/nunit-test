This is just a test repository I'm using to test deploying projects while I'm learning how to use and configure Nginx Unit.

It's not going to do anything fancy, and I'll delete it at some point.

Log of commands issued to test server:

```bash
#Install test app
mkdir -p /www
chown 
#One liner pre-setup
curl -sL 'https://unit.nginx.org/_downloads/setup-unit.sh' | sudo -E bash
#Setup
apt install unit unit-php
#Install config
#TODO: Use jq to loop over each section, set individually via API, to allow existing config to be present.
curl -X PUT \
	--data-binary @config/nginx.json \
	--unix-socket /var/run/control.unit.sock \
	"http://localhost/config
```
