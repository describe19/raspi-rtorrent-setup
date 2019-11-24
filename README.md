# RasPi rTorrent setup
## Some initial setup

```bash
sudo apt-get update
sudo apt-get upgrade
sudo nano /etc/apt/sources.list
sudo apt-get install -y subversion screen curl git nano
```
And add ``` main contrib non-free ``` to each source.


## Dependencies
Apache2 for webui (rutorrent)
PHP for rutorrent 
```bash
sudo apt-get install -y apache2 apache2-utils libxmlrpc-core-c3 libapache2-mod-php
```





## Sources:
https://no.help/rTorrent-ruTorrent-Seedbox-Guide.php

https://terminal28.com/how-to-install-and-configure-rutorrent-rtorrent-debian-9-stretch/

https://github.com/rakshasa/rtorrent/wiki/Config-Guide
