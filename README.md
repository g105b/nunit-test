This repository holds the test code I'm writing to learn how to set up and deploy Nginx Unit.

Current status: provide the bash script below with the name of the application and the application's domain, and this repository will be deployed to the server with all dependencies handled and TLS set up.

Feel free to reference the way I've set any of this up, fork it, steal it, whatever.

Set up the entire server in a single script:

```bash
set -e
app_name="$1"
domain="$2"
if [[ "$domain" != *"."* ]]
then
	echo "Incorrect usage. Arguments should be: "
	echo "1. app_name"
	echo "2. domain"
	exit 1
fi

# Install server requirements
export DEBIAN_FRONTEND=noninteractive
curl -sL 'https://unit.nginx.org/_downloads/setup-unit.sh' | bash
apt-get install -y unit unit-php git unzip jq php-{curl,xml,mbstring,zip}
echo "Done apt-get"

# Install Composer
wget https://raw.githubusercontent.com/composer/getcomposer.org/76a7060ccb93902cd7576b67264ad91c8a2700e2/web/installer -O - -q | php -- --quiet
mv composer.phar /usr/local/bin/composer

# Configure unit user
mkdir -p /var/www
chown unit. /var/www
systemctl stop unit
usermod -d /var/www unit
mkdir -p /var/log/unit
systemctl start unit

# Log in as unit user to configure local commands
sudo -u unit bash <<"UNIT_BASH" -s -- $app_name
app_name=$1
cd
mkdir html
# Set up "gt" command
composer global require phpgt/gtcommand
cat << EOF >> ~/.bashrc
export PATH="$PATH:~/.config/composer/vendor/bin"
EOF
export PATH="$PATH:~/.config/composer/vendor/bin"

# Set up "npm" command, along with "webpack" and "sass"
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
cat << EOF >> ~/.bashrc
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
EOF
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install v16
npm install --global webpack-cli sass

# Install test app
git clone https://github.com/g105b/nunit-test $app_name
cd $app_name
composer install
php vendor/phpgt/webengine/setup.php
gt build
UNIT_BASH

# Configure web server
useradd --no-create-home unit-$app_name
usermod -L unit-$app_name
usermod -aG unit-$app_name unit
 
tmp=/tmp/nginx-unit-cfg
cfgPath=/var/www/$app_name/config/nginx.json

# A convenience function wrapping the curl configuration
# Pass two/three arguments: 
# 1. The JSON string containing configuration, e.g. { "actions": { "return": 404 }}
# 2. The path to the configuration, e.g. routes.test-app
# 3. Optional - The section of configuration to apply, e.g. certificates (default: config)  
unit_cfg () {
	cfg=$1
	path=$2
	section=${3-config}
	echo "$cfg" > $tmp
	curl -X PUT \
		--data-binary @$tmp \
		--unix-socket /var/run/control.unit.sock \
		"http://localhost/$section/$path"
	rm $tmp
}

# Handle generation of TLS certificate
snap install core
snap refresh core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
certbot certonly --noninteractive --standalone --agree-tos --email $app_name.g105b.com.certbot@g105b.com -d $domain
bundle=$(cat /etc/letsencrypt/live/$domain/fullchain.pem /etc/letsencrypt/live/$domain/privkey.pem)

# Set up Nginx Unit configuration areas
unit_cfg "$bundle" "$app_name-certbot" "certificates"
cfg=$(jq '.access_log' "$cfgPath")
unit_cfg "$cfg" "access_log"

config_applications=$(jq -r '.applications | to_entries[] | .key' "$cfgPath")
while IFS= read -r application
do
	cfg=$(jq ".applications.\"$application\"" "$cfgPath")
	unit_cfg "$cfg" "applications/$application"
done <<< $config_applications

unit_cfg "{}" "routes"
config_routes=$(jq -r '.routes | to_entries[] | .key' "$cfgPath")
while IFS= read -r route
do
	cfg=$(jq ".routes.\"$route\"" "$cfgPath")
	unit_cfg "$cfg" "routes/$route"
done <<< $config_routes

config_listeners=$(jq -r '.listeners | to_entries[] | .key' "$cfgPath")
while IFS= read -r listener
do
	cfg=$(jq ".listeners.\"$listener\"" "$cfgPath")
	unit_cfg "$cfg" "listeners/$listener"
done <<< $config_listeners

echo "Completed setting up: $domain"
```

### Certbot

At setup time, the first certificate is generated with the name `$app_name-certbot`, and is done via the standalone webserver of certbot. The name is needed, as the listener on port 443 refers to this certificate name. The standalone webserver is needed at this point as there is nothing configured to listen on any ports yet.

For renewals, the standalone server can't be used without having to temporarily turn off Nginx Unit - this would cause downtime so instead, renewal challenge files are placed directly in the application's web root and served by the route's "share" configuration.

The following script should be executed periodically, such as in a monthly crontab, to handle renewals:

TODO: This section needs to work with multiple certificates, explained in [issue #2](https://github.com/g105b/nunit-test/issues/2)

```bash
certbot certonly \
	--agree-tos \
	--email $domain.certbot@g105b.com \
	--webroot \
	--webroot-path /var/www/$app_name/www/ \
	--deploy-hook /var/www/$app_name/config/certbot-hook.bash \
	-n \
	-d $domain

# Add the new certificate, date stamped
date_stamp=$(date "+%F")
tmp=/tmp/nginx-unit-cfg
cat /etc/letsencrypt/live/testing.g105b.com/fullchain.pem /etc/letsencrypt/live/testing.g105b.com/privkey.pem > $tmp
curl --request PUT --data-binary @$tmp --unix-socket /var/run/control.unit.sock "http://localhost/certificates/testing-certbot-$date_stamp"
# Set the listener to use the new certificate
curl --request PUT --data "\"testing-certbot-$date_stamp\"" --unix-socket /var/run/control.unit.sock "http://localhost/config/listeners/*:443/tls/certificate"
```
