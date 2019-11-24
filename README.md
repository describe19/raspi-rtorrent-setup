# RasPi rTorrent setup
Mostly stolen directly from: https://no.help/rTorrent-ruTorrent-Seedbox-Guide.php

With some fixes (and not compiling anything from source).
## Some initial setup

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install -y subversion screen curl git nano
sudo nano /etc/apt/sources.list
```
And add ``` main contrib non-free ``` to each source.

## rtorrent
#### Install
Before running check if there is a newer libtorrent: ```apt search libtorrent```
```bash
sudo apt-get install -y install rtorrent libtorrent20 libxmlrpc-core-c3
```

#### Configure
Run the following to get the official default config:
```bash
curl -Ls "https://raw.githubusercontent.com/wiki/rakshasa/rtorrent/CONFIG-Template.md" \
    | sed -ne "/^######/,/^### END/p" \
    | sed -re "s:/home/USERNAME:$HOME:" >~/.rtorrent.rc
mkdir -p ~/rtorrent/
```
You may wish to modify the directories defined at the beginning and also the "## Tracker-less torrent and UDP tracker support
"

Check rtorrent runs ok

Now add the following to the config:
```
scgi_port = 127.0.0.1:5000
system.daemon.set = true
```
scgi_port allows rutorrent to communicate with rtorrent and system.daemon.set = true runs rtorrent without the interface.

Create a system service to start rtorrent on boot:

```bash
sudo nano /etc/systemd/system/rtorrent.service
```

Paste the following (replacing USERNAME, and changing the location of the session if you modified it from the default):
```
[Unit]
Description=rTorrent Daemon
After=network.target

[Service]
[Unit]
Description=rTorrent Daemon
After=network.target

[Service]
Type=simple
KillMode=process
User=USERNAME
ExecStartPre=/bin/bash -c "if test -e /home/USERNAME/rtorrent/.session/rtorrent.lock && test -z pidof rtorrent; then rm -f /home/nath/rtorrent/.session/rtorrent.lock; fi"
ExecStart=/usr/bin/rtorrent
WorkingDirectory=/home/nath/rtorrent
Restart=on-failure
[Install]
WantedBy=multi-user.target
```
[source](https://github.com/rakshasa/rtorrent/issues/844)

## ruTorrent
### Apache2, PHP and SSL
ruTorrent is just a website/webpage, hence a webserver like apache2 or nginx is needed to host it.
#### Install apache2:
```bash
sudo apt-get install -y apache2 apache2-utils  libapache2-mod-php
```

#### Configure apache2:
```bash
sudo a2enmod auth_digest ssl reqtimeout
sudo nano /etc/apache2/apache2.conf
```
And edit or add:
```
Timeout 30
ServerSignature Off
ServerTokens Prod
```
"```ServerTokens Prod``` will prevent Apache from reporting its version number, and ```ServerSignature Off``` will disable the server signature displayed in Apache error messages. Small steps to harden your system."

Restart apache2
```bash
sudo service apache2 restart
```

Confirm Apache2 and PHP are working:

```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/html/info.php
```

Open a browser and go to <your.ip.address>/info.php

You should see a large table of info - take note of the ```Loaded Configuration file```.

#### Create an SSL certificate for https access.
When prompted for CN (common name) enter the hostname (usually raspberrypi unless you've changed it).
```bash
sudo mkdir /etc/apache2/ssl
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/apache2/ssl/apache.pem -out /etc/apache2/ssl/apache.pem
sudo chmod 600 /etc/apache2/ssl/apache.pem
```
Configure apache access:
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

And paste the following (replacing <your.ip.address>)
```
<VirtualHost *:80>
ServerAdmin webmaster@localhost

DocumentRoot /var/www/html
Redirect permanent / https://<your.ip.address>/
Redirect permanent /rutorrent https://<your.ip.address>/rutorrent
</VirtualHost>

<VirtualHost _default_:443>
ServerAdmin webmaster@localhost
ServerName <your.ip.address>:443

SSLEngine on
SSLCertificateFile /etc/apache2/ssl/apache.pem

DocumentRoot /var/www/html/
<Directory />
  Options FollowSymLinks
  AllowOverride All
</Directory>
<Directory /var/www/html/>
  Options Indexes FollowSymLinks MultiViews
  AllowOverride None
  Order allow,deny
  allow from all
</Directory>

ErrorLog ${APACHE_LOG_DIR}/error.log

# Possible values include: debug, info, notice, warn, error, crit, alert, emerg.
LogLevel warn

CustomLog ${APACHE_LOG_DIR}/access.log combined

<Location /rutorrent>
  AuthType Digest
  AuthName "rutorrent"
  AuthDigestDomain /var/www/html/rutorrent/ https://<your.ip.address>/rutorrent
  AuthDigestProvider file
  AuthUserFile /etc/apache2/.htpasswd
  Require valid-user
  SetEnv R_ENV "/var/www/html/rutorrent"
</Location>
</VirtualHost>
```
Now:
```bash
sudo a2ensite default-ssl
sudo nano /etc/apache2/ports.conf
```
And make sure it looks something like:
```
Listen 80

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>
```

```bash
sudo service apache2 restart
```

Now check that going to <your.ip.address> in a browser redirects to https.
#### Optional: Modify PHP config
Uploading many torrents to rutorrent will fail unless you allow as many file uploads in the php config.
Open the loaded php config you noted earlier (or create the info.php file again like above and have a look, delete it again afterwards).
Search for the following lines and change them to whatever you like:

```upload_max_filesize = 64M
max_file_uploads = 200
post_max_size = 128M
```

### ruTorrent (finally)
#### Dependencies
```bash
sudo apt-get install -y zip unzip zlib1g-dev ffmpeg mediainfo unrar 
```

#### Download ruTorrent + latest plugins
```bash
cd /var/www/html
sudo git clone https://github.com/Novik/ruTorrent.git rutorrent
sudo rm -r rutorrent/plugins
sudo svn checkout https://github.com/Novik/ruTorrent/trunk/plugins rutorrent/plugins
```

#### Configure ruTorrent
```bash
sudo chown -R www-data:www-data rutorrent
sudo chmod -R 755 rutorrent
sudo nano rutorrent/conf/config.php
```
and edit the following, changing directories to what you want:
```
	$log_file = '/tmp/rutorrent_errors.log';
	$topDirectory = '/home/downloads/';
	$pathToExternals = array(
		"php" => '/usr/bin/php',
		"curl" => '/usr/bin/curl',
		"gzip" => '/bin/gzip',
		"id" => '/usr/bin/id',
		"stat" => '/usr/bin/stat',
	);
  ```
Configure plugins:
```bash
sudo rm -f /var/www/html/rutorrent/conf/plugins.ini
sudo nano /var/www/html/rutorrent/conf/plugins.ini
```

and paste from [here](https://no.help/articles/seedbox/plugins.ini.txt).

Now for auth (use a different username/password (or not you dont have to do what I say)):
```bash
sudo htdigest -c /etc/apache2/.htpasswd rutorrent USERNAME
```

Now go to <your.ip.address>/rutorrent

## TODO
### autodl-irssi




## Sources:
https://no.help/rTorrent-ruTorrent-Seedbox-Guide.php

https://terminal28.com/how-to-install-and-configure-rutorrent-rtorrent-debian-9-stretch/

https://github.com/rakshasa/rtorrent/wiki/CONFIG-Template

https://github.com/rakshasa/rtorrent/issues/844
