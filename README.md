This is just a test repository I'm using to test deploying projects while I'm learning how to use and configure Nginx Unit.

It's not going to do anything fancy, and I'll delete it at some point.

Log of commands issued to test server:

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

snap install core
snap refresh core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
certbot certonly --noninteractive --standalone --agree-tos --email $app_name.g105b.com.certbot@g105b.com -d $domain

bundle=$(cat /etc/letsencrypt/live/$domain/fullchain.pem /etc/letsencrypt/live/$domain/privkey.pem)
unit_cfg "$bundle" "$app_name-certbot" "certificates"

cfg=$(jq '.access_log' "$cfgPath")
unit_cfg "$cfg" "access_log"

config_applications=$(jq -r '.applications | to_entries[] | .key' "$cfgPath")
while IFS= read -r application
do
	cfg=$(jq ".applications.\"$application\"" "$cfgPath")
	unit_cfg "$cfg" "applications/$application"
done <<< $config_applications

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
```

### Certbot

At setup time, the first certificate is generated with the name `$app_name-certbot`, and is done via the standalone webserver of certbot. The name is needed, as the listener on port 443 refers to this certificate name. The standalone webserver is needed at this point as there is nothing configured to listen on any ports yet.

For renewals, the standalone server can't be used without having to temporarily turn off Nginx Unit - this would cause downtime so instead, renewal challenge files are placed directly in the application's web root and served by the route's "share" configuration.
