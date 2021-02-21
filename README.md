I came to know about Nextcloud about a year or so back, but then I could not believe it would come to grow to such an extent as we see today.

Nextcloud comes with every kind of tool that you may expect from a Google Cloud or Onedrive. Emails, Calendars, Chatting, Uncompressed Photo storage and sharing, so on and so forth. And like Wordpress new apps can be added thereby extending the suite of functionalities it has to offer.

And of course Nextcloud is GDPR and HIPAA compliant, and if you are self-hosting it, there is absolute privacy.

So I finally decided to give it a go.

I had some requirements though from the whole setup.

1. I wanted it to be set up on a domain like: cloud.you.com
2. It should have full A+ SSL
3. Should not cost me more than 3-4 EUR a month
4. I should be able to use the server for my other small projects.
5. Send and Receive email
6. I should be able to activate the Face Recognition applet which needs around 2GB of RAM
7. Should be able to add Backblaze B2 object storage 

And while I was at it, installing Nextcloud manually on a personal VPS server, I noted everything down, so as to be able to share the experience/knowledge. You do not need an engineering degree, or have heard the word PHP before. You can blindly follow the steps, I would of course explain each one of them. But if you are a pro, I wish you can improve this guide in some way. Point out mistakes or redundancies if found or suggest improvements too.

## Getting a personal Cloud.
Depending on where you are right now, I would suggest getting a server that's located very close to you. That keeps the latencies low.

Of course if you do not mind the few ms of latency, the world is your playground.

### For India
Digital Ocean Bangalore could be great, a server with 1GB RAM costs $5/month.

### For USA
There are several locations from vultr.com, and I believe no one can beat vultr at pricing. You can get a server with 512MB at $3.5/month

There is something more to US, you can get a forever free server with 512MB RAM from Google Cloud, and from Amazon you can get a similar offer for one year free of cost.

### For Germany
And this is a top secret, not many people know about Hetzner. Hetzner has locations in Nurenberg, Falkenstein and Helsinki. Which are very close to Berlin, for me. Also the pricing ist unschlagbar: 2.96 EUR/month for an Ubuntu server with 2GB RAM!

## First things first

### Update
The first things to do on a new server is always update the system:

```
sudo apt-get update

sudo apt-get upgrade
```

### Install Swapfile
Nextcloud says they need only about 128 MB of RAM, but that's like saying a snake needs to eat only twice a year. Of course more RAM makes it possible to do multi tasking, faster processing etc. As my plan includes the Face Recognition applet I need to eke out as much as possible from the system.

Given that I have 20GB SSD space I can afford to give away 3GB for swap.

#### Allocating a 3GB /swapfile on root
```
sudo fallocate -l 3G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Now if you do `free -h` you would see there is a 3GB swap added to available memory.

#### We also need to make sure the swapfile is loaded at every reboot.

Create a backup of fstab
```
sudo cp /etc/fstab /etc/fstab.bak
```
Add the swapfile to fstab
```
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Reboot `reboot` and check memory `free -h` to confirm

### Update timezone
Most VPS come with UTC as default timezone, this might create some errors if you plan to use your email service with Nextcloud and you are not on the UTC timezone.

Update time zone like so
```
timedatectl set-timezone "Europe/Berlin"
```

If you are not sure about your timezone you can do `timedatectl list-timezones` to get a list of them. However, for India it is `"Asia/Kolkata"`

## Installing required software

### Install Unzip
```
sudo apt-get install unzip
```

### Install Server: Nginx
Nextcloud recommends Apache2, but I did burn my fingers there. So no more Apache. Nginx FTW!

```
sudo add-apt-repository ppa:ondrej/nginx
sudo apt-get install nginx -y
sudo apt-get update
```

### Install Database: MariaDB
Nextcloud recommends MariaDB. I have no opinion here, this would not affect me.

```
sudo apt install mariadb-server -y
sudo systemctl enable mariadb
sudo apt-get update
```

### Install Controller: PHP
Nextcloud recommends latest PHP 7 i.e. PHP 7.4

```
sudo add-apt-repository ppa:ondrej/php
sudo apt install php-fpm php-curl php-cli php-mysql php-gd php-common php-xml php-json php-intl php-pear php-imagick php-dev php-common php-mbstring php-zip php-soap php-bz2 -y
sudo apt-get update
```

### Install SSL Manager: Let's Encrypt
It needs some Python support

```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get install software-properties-common
sudo apt-get install python3-pyasn1
sudo apt-get install python-certbot-nginx
sudo apt-get update
```

## Configuring required software
Now that all installations are done, we would need to configure them to work together.

### Securing MySQL
MariaDB does not come with a password set, so we need to do first set up a password.

```
sudo mysql_secure_installation
```

1. First it would ask you to enter root password, but simply press Enter to continue.
2. Then it would ask `Set root password? [Y/n]`, press Y and set it up.
3. `Remove anonymous users? [Y/n]` Y << Type Y to remove anonymous users
4. `Disallow root login remotely? [Y/n]` Y  << Type Y to disable root login remotely
5. `Remove test database and access to it? [Y/n]` Y << Type Y to remove test database
6. `Reload privilege tables now? [Y/n]` Y << Type Y

### Securing PHP
PHP comes with this weird option turned on. `fix_pathinfo` basically corrects the URL of a file if you provide an incorrect address. Which is very insecure, as someone might be trying to guess things around.

#### You can turn `fix_pathinfo` off by doing:
```
sudo sed -i s/\;cgi\.fix_pathinfo\s*\=\s*1/cgi.fix_pathinfo\=0/ /etc/php/7.4/fpm/php.ini
sudo sed -i s/\;cgi\.fix_pathinfo\s*\=\s*1/cgi.fix_pathinfo\=0/ /etc/php/7.4/cli/php.ini
```

#### Restart PHP
```
sudo systemctl restart php7.4-fpm
```

### Add PHP process
Assuming you would like to host the setup on https://cloud.you.com

#### Keep a backup of the existing `www.conf` file
```mv /etc/php/7.4/fpm/pool.d/www.conf{,.bak}```

#### Create new process
```sudo nano /etc/php/7.4/fpm/pool.d/cloud.you.com.conf```

#### Put this in the file
```
[cloud.you.com]
user = www-data
group = www-data
listen = /run/cloud.you.com-fpm.sock
listen.owner = www-data
listen.group = www-data
pm = ondemand
pm.process_idle_timeout = 30s
pm.max_requests = 512
pm.max_children = 30
```

Press Ctrl+X to save, type Y to confirm, and then press Enter, remember these steps for editing files with nano.

#### Restart PHP
```
sudo systemctl restart php7.4-fpm
```

### Create web directory
```
mkdir -p /var/www/cloud.you.com/{files,log}
```

Inside `files` add `index.php` with a line of code
```
echo "<?php echo 'Hello World';" >> /var/www/cloud.you.com/files/index.php
```

Inside the `log` folder create blank files
```
touch access.log error.log
```

### Configure Nginx
#### Create a new site
```sudo nano /etc/nginx/sites-available/cloud.you.com```

Add this to the file
```
server {
	listen 80;
	server_name cloud.you.com;
	root /var/www/cloud.you.com/files;
	index index.php index.html index.htm;

	access_log /var/www/cloud.you.com/log/access.log;
	error_log  /var/www/cloud.you.com/log/error.log notice;

	location ~ \.php$ {
		try_files $uri =404;
		fastcgi_split_path_info ^(.+\.php)(/.+)$;
		include fastcgi_params;
		fastcgi_read_timeout 300;
		fastcgi_intercept_errors on;
		fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
		fastcgi_pass unix:/run/cloud.you.com-fpm.sock;
	}
}
```

#### To enable the site
Create a shortcut of the file in `/etc/nginx/sites-enabled`
```
ln -s /etc/nginx/sites-available/cloud.you.com /etc/nginx/sites-enabled/
```

#### Restart nginx
```service nginx restart```

### Configure Let's Encrypt and SSL
#### Request certificate
```
sudo certbot certonly --webroot -w /var/www/cloud.you.com/files/ -d cloud.you.com
```
Read the insructions they would ask for some consent, nothing technical.

### Set up Nginx to use SSL
#### Creating Diffie-Hellman prime
Create a strong Diffie-Hellman parameter
```
openssl dhparam -out /etc/nginx/dhparam.pem 4096
```

This may take some time, it is equally fine if you create the file 

```
sudo nano /etc/nginx/dhparam.pem
```
and paste this in there
```
-----BEGIN DH PARAMETERS-----
MIICCAKCAgEApFPvfYTqaFFctEcnBiZh9sXs6k32Jva+/MhmgWqJ4bOF6EDiY3tF
h4a57ktzEyEbkeoULDni8MGyqtD0REA5uA8d9YMA83A+w91DqPnRtM6vGAzGIXuN
lvMODQtpHJMbOCACLirJlwpSeKTqzGvYmXmLtP59ZBJsf6Aun0gpvTlYqN+nbnVt
fRQsA7jTAadB+kYB7olVQar4DUQLKO1wm4Up/5pNt7ui3GLex7/+Vf1K50vfJiX3
0mRRYO7gCC6g55nh089LxQ9wuIA8nFJouS+0+vBu02oFHNiukGr421NC5jVYRvH3
uZiVitq4A9PzNgg2g8d8/EXSbg7C1bUuF1JQQ63CxJirhxH2+YvzrAODUcBhzDGr
rSaTG8y4yujezgRhu05JNTwbfRs3toOLEncZ3lagKQj5cWJIBisJpFwNCAlBcVfM
AhXJ5ldF8P4+nJoOmtoZ8I7PF5LwrDjPdk4jkYJGDiA/w8oyqhzouh1D3dQBtdZd
2kPBfBsl7stY2sbUl4wzzMgPuaBxDc+aJl6EkjSjz8IrVP2jyzx9JSyT77KDcxZW
5o9JrDrRuWxqnByHrP9osOe2WexgEEInnsW/JT+AG/FlPCA+ieXA5NWKqd7+Jpiy
ZoIwzxyAmsYTTsP+aB9nl2GaFKQN+kcxSIxVXHVRMAuLAqrNPNQhpnMCAQI=
-----END DH PARAMETERS-----
```

#### Configure Nginx
Now open `/etc/nginx/nginx.conf`

```
sudo nano /etc/nginx/nginx.conf
```
and replace the SSL section with this:

```
	ssl_dhparam /etc/nginx/dhparam.pem; 
	ssl_protocols TLSv1.2;
	ssl_prefer_server_ciphers on; 
	ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
	ssl_ecdh_curve secp384r1; 
	ssl_session_timeout  10m;
	ssl_session_cache shared:SSL:10m;
	ssl_session_tickets off; 
	ssl_stapling on; 
	ssl_stapling_verify on;
	resolver 8.8.8.8 8.8.4.4 valid=300s;
	resolver_timeout 5s; 
	add_header Strict-Transport-Security "max-age=63072000; includeSubDomains";
	add_header X-Frame-Options DENY;
	add_header X-Content-Type-Options nosniff;
	add_header X-XSS-Protection "1; mode=block";
```

open the site configuration
```
sudo nano /etc/nginx/sites-available/cloud.you.com
```

and replace the entire text with this

```
upstream php-handler {
	#server 127.0.0.1:9000;
	server unix:/run/cloud.you.com-fpm.sock;
}

server {
	listen 80;
	listen [::]:80;
	server_name cloud.you.com;
	# enforce https
	return 301 https://$server_name$request_uri;
}

server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	server_name cloud.you.com;
	root /var/www/cloud.you.com/files;
	index index.php index.html index.htm;

	access_log /var/www/cloud.you.com/log/access.log;
	error_log  /var/www/cloud.you.com/log/error.log notice;

	ssl_certificate /etc/letsencrypt/live/cloud.you.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/cloud.you.com/privkey.pem;


	location = /.well-known/carddav {
	  return 301 $scheme://$host/remote.php/dav;
	}

	location = /.well-known/caldav {
	  return 301 $scheme://$host/remote.php/dav;
	}

	# set max upload size
	client_max_body_size 512M;
	fastcgi_buffers 64 4K;

	# Enable gzip but do not remove ETag headers
	gzip on;
	gzip_vary on;
	gzip_comp_level 4;
	gzip_min_length 256;
	gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
	gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

	location / {
		rewrite ^ /index.php$uri;
	}

	location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
		deny all;
	}
	location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
		deny all;
	}

	location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+)\.php(?:$|/) {
		fastcgi_split_path_info ^(.+\.php)(/.*)$;
		include fastcgi_params;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		fastcgi_param PATH_INFO $fastcgi_path_info;
		fastcgi_param HTTPS on;
		#Avoid sending the security headers twice
		fastcgi_param modHeadersAvailable true;
		fastcgi_param front_controller_active true;
		fastcgi_pass php-handler;
		fastcgi_intercept_errors on;
		fastcgi_request_buffering off;
	}

	location ~ ^/(?:updater|ocs-provider)(?:$|/) {
		try_files $uri/ =404;
		index index.php;
	}

	# Adding the cache control header for js and css files
	# Make sure it is BELOW the PHP block
	location ~ \.(?:css|js|woff|svg|gif)$ {
		try_files $uri /index.php$uri$is_args$args;
		add_header Cache-Control "public, max-age=15778463";
		# Add headers to serve security related headers (It is intended to
		# have those duplicated to the ones above)
		# Before enabling Strict-Transport-Security headers please read into
		# this topic first.
		# add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;";
		#
		# WARNING: Only add the preload option once you read about
		# the consequences in https://hstspreload.org/. This option
		# will add the domain to a hardcoded list that is shipped
		# in all major browsers and getting removed from this list
		# could take several months.
		add_header X-Content-Type-Options nosniff;
		add_header X-XSS-Protection "1; mode=block";
		add_header X-Robots-Tag none;
		add_header X-Download-Options noopen;
		add_header X-Permitted-Cross-Domain-Policies none;
		# Optional: Don't log access to assets
		access_log off;
	}

	location ~ \.(?:png|html|ttf|ico|jpg|jpeg)$ {
		try_files $uri /index.php$uri$is_args$args;
		# Optional: Don't log access to other assets
		access_log off;
	}
}
```

#### Restart nginx
```service nginx restart```

### Download Nextcloud
#### Goto the web directory
```
cd /var/www/cloud.you.com
```

#### Download Nextcloud and unzip
```
wget https://download.nextcloud.com/server/releases/nextcloud-20.0.4.zip
unzip nextcloud-20.0.4.zip
```

#### Move Nextcloud to files
```mv nextcloud files```

#### Create another directory for documents
```mkdir data```

#### Make them writable by PHP
```
sudo chown -R www-data:www-data data
```

```
sudo chown -R www-data:www-data files
```

### Setup Database for Nextcloud
```sudo mysql```

Here we would be specifying the user name and the password for the database, this would be used by Nextcloud app to connect to the database. 
It's also therefore important to keep the user name and password super difficult.
It's totally fine if you forget it later, but for now store the user and password somewhere.

```
create database nextcloud_db;
create user clouduser@localhost identified by 'cloudpass';
grant all privileges on nextcloud_db.* to clouduser@localhost identified by 'cloudpass';
flush privileges;
exit;
```

## Configuring Nextcloud
Aaand we are done!

Visit: `https://cloud.you.com` now

It would ask you to provide the username, password for the admin user and for the database.

Input all the information correctly and submit.

## Face Recognition and Backblaze setup
To be continued...
