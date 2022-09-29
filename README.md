This is just a test repository I'm using to test deploying projects while I'm learning how to use and configure Nginx Unit.

It's not going to do anything fancy, and I'll delete it at some point.

Log of commands issued to test server:

```bash
# Install server requirements
curl -sL 'https://unit.nginx.org/_downloads/setup-unit.sh' | sudo -E bash
apt install unit unit-php git unzip php{xml,mbstring,zip}

# Install Composer
wget https://raw.githubusercontent.com/composer/getcomposer.org/76a7060ccb93902cd7576b67264ad91c8a2700e2/web/installer -O - -q | php -- --quiet
mv composer.phar /usr/local/bin/composer

# Configure unit user
mkdir -p /var/www
chown unit. /var/www
systemctl stop unit
usermod -d /var/www unit
systemctl start unit

# Log in as unit user to configure local commands
sudo -u unit /bin/bash
cd
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
[ -s "\$NVM_DIR/nvm.sh" ] && \. "\$NVM_DIR/nvm.sh"
EOF
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install v16
npm install --global webpack-cli sass

# Install test app
git clone https://github.com/g105b/nunit-test myapp
cd myapp
composer install
gt build
exit

# Configure web server
#TODO: Use jq to loop over each section, set individually via API, to allow existing config to be present.
curl -X PUT \
	--data-binary @/var/www/myapp/config/nginx.json \
	--unix-socket /var/run/control.unit.sock \
	"http://localhost/config
```
