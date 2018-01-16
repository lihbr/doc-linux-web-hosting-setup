# Linux web hosting setup (Debian)

This is just a basic reminder page for setting up a linux server for web hosting, please note that this is my first itteration of it and this page is just basicaly here to help me reming of some basics and it may exist better way to do things. Anyway, feel free to suggest anything !

# Table of Contents

1. [Connection](#connection)
2. [Setting up root account](#setting-up-root-account)
3. [Managing users](#managing-users)
    1. [Creating new user](#creating-new-user)
    2. [Managing user's groups](#managing-users-groups)
4. [Setting up web services](#setting-up-web-services)
5. [Apache configuration](#apache-configuration)
    1. [Default mods](#default-mods)
    2. [Moving default directory](#moving-default-directory)
    3. [Creating new Virtual Host (aKa new website)](#creating-new-virtual-host-aka-new-website)
    4. [Managing active website](#managing-active-website)
    5. [Restarting Apache service (to load new configuration)](#restarting-apache-service-to-load-new-configuration)
6. [MySQL configuration](#mysql-configuration)
    1. [Define root password](#define-root-password)
7. [Adding SSL certificate](#adding-ssl-certificate)


# Connection

On command prompt:
```
$ ssh root@SERVER_IP_ADDRESS
```

Then it'll ask for password.

# Setting up root account

First thing to do is to change the root password, to do that type:
```
$ passwd
```
Then it'll ask you for new password

Next we'll secure connection to root through Public Key Authentification to do that you'll need to generate an SSH key pair on a local bash (i.e. git bash on windows) and follow step after this:
```
$ ssh-keygen
```

Once you're done you'll need to copy the public key onto your server (still on local bash):
```
$ ssh-copy-id -i ~/.ssh/SSH_KEY_NAME root@SERVER_IP_ADDRESS
```
It'll ask you for the account password (you can do the same to secure another account through PKA).

Then you can log on you server easily using your private key, you can add multiple private key to one user so to check who can access a user with a private key just type:
```
$ cat /USER_NAME/.ssh/authorized_keys
```

In order to remove a key from a user just edit authorized_keys with vi text editor and remove line associated with it.

Once you can login to root through a private key you can disable root auth through login to do so edit the ssh config with:
```
$ vi /etc/ssh/sshd_config
```

And change `PermitRootLogin yes` to `PermitRootLogin without-password`, once it's done you'll need to restart the SSH service:
```
$ service ssh restart
```

# Managing users

## Creating new user

For everyday use you may prefer to work with a *regular* user, in order to create it follow this:
```
$ adduser USER_NAME
```
Then it'll ask you for the user password and other info related to the new user (that you can skip).

## Managing user's groups

Adding user to group:
```
$ adduser USER_NAME GROUP_NAME
```

Then the user should be in the chosen group, you can check that with this command:
```
$ grep 'GROUP_NAME' /etc/group
```

To remove user from a group type:
```
$ deluser USER_NAME GROUP_NAME
```

In order to list all existing group perform this command:
```
$ cut -d: -f1 /etc/group
```

# Setting up web services

Install AMP stack through this:
```
$ apt-get install apache2 php mariadb-server mariadb-client libapache2-mod-php7.0 php7.0 php7.0-mysql php7.0-curl php7.0-json php7.0-gd php7.0-mcrypt php7.0-mbstring php7.0-xml php7.0-zip
```

By default all of these should be launch on system boot, you can check what is launched on boot by checking `/etc/init.d` folder.

Also to check if a service is running prompt the following:
```
$ service SERVICE_TO_CHECK_(i.e. apache2) status
```

All basic web services should be good now.

# Apache configuration

## Default mods
Those are the default mods that i use to run Apache, you may need to install them first:
```
$ a2enmod deflate
$ a2enmod headers
$ a2enmod rewrite
$ a2enmod ssl
```

## Moving default directory

We'll see how to move the default directory (`/var/www`) to a new one.

First we'll need to edit some apache default configuration file:
```
$ vi /etc/apache2/sites-enabled/default.conf
```

Update it to something like this:
```apache
<VirtualHost *:80>
  ServerAdmin webmaster@localhost
  DocumentRoot NEW_PATH_TO_WEBSITE
...
```

## Creating new Virtual Host (aKa new website)
I think it's better to create a new `.conf` file in `/etc/apache2/sites-available` for each new website you'll host on you server.

So first create a new file at `/etc/apache2/sites-available`:
```
$ touch SITE_NAME.conf
```

Then edit it with:
```
$ vi SITE_NAME.conf
```

Here's an example basic configuration for a subdomain website:
```apache
<VirtualHost *:80>
  ServerAdmin webmaster@localhost

  ServerName sub.domain.tld
  ServerAlias alt.sub.domain.tld

  DocumentRoot /path/to/website

  <Directory /path/to/website/>
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    Require all granted

    # Add only if you have ssl certificate setup with SITE_NAME-ssl.conf
    RewriteEngine on
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,QSA,R=permanent]
    
    # compression with MOD_DEFLATE
    AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css text/javascript application/atom+xml application/rss+xml application/xml application/javascript
    # for proxys user
    Header append Vary User-Agent env=!dont-vary
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Once you're done with this new website configuration you need to reload Apache [(see below)](#managing-active-website), ofc new website configuration will work only if your DNS is up to date.

To delete a website configuration use `rm` command (be careful)

## Managing active website

To enable a website:
```
$ a2ensite CONF_NAME_WITHOUT_.CONF
```

To disable a website:
```
$ a2dissite CONF_NAME_WITHOUT_.CONF
```

## Restarting Apache service (to load new configuration)

To check if your new configuration is correct you can prompt the following:
```
$ apachectl configtest
```

If everything is ok, then you'll need to restart Apache:
```
$ service apache2 restart
```

If something messed up you can check log here:
```
$ tail /var/log/apache2/error.log
```

# MySQL configuration

## Define root password

Enter configuration with:
```
$ mysql -u root
```

Then setup root password with:
```sql
 > UPDATE mysql.user SET Password = PASSWORD('PASSWORD')
-> WHERE User = 'root';
```
Then apply change with:
```sql
 > FLUSH PRIVILEGES;
```

# Adding SSL certificate

We'll see how to do so with Let's Encrypt.

First you need to have git installed, if it's not the case then:
```
$ apt-get install git-all
```

Once it's do the following:
```
$ cd /opt && git clone https://github.com/certbot/certbot.git && ln -s  certbot/certbot-auto /usr/bin
$ /opt/certbot/certbot-auto --apache
```

If something goes wrong related to satisfying the CA try this after installation:
```
$ /opt/certbot/certbot-auto  --authenticator standalone --installer apache --pre-hook "service apache2 stop" --post-hook "service apache2 start"
```

Then Certbot should have created your certificate, and you want them to renew automatically, to do so create a cron:
```
$ crontab -e
```

And add to it a line like this (each day at 3:19):
```
19 3 * * * /opt/certbot/certbot-auto renew
```

To view existing certificates:
```
$ /opt/certbot/certbot-auto certificates
```

In order to activite https on your site you need to create a `SITE_NAME-ssl.conf` in the Apache `sites-available` folder for each of your site. Here's an example:
```apache
<VirtualHost *:80>
  ServerAdmin webmaster@localhost
  
  ServerName sub.domain.tld
  ServerAlias alt.sub.domain.tld
  
  DocumentRoot /path/to/website
  
  <Directory /path/to/website/>
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    Require all granted
    RewriteEngine on
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,QSA,R=permanent]
    
    # compression with MOD_DEFLATE
    AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css text/javascript application/atom+xml application/rss+xml application/xml application/javascript
    # for proxys user
    Header append Vary User-Agent env=!dont-vary
  </Directory>
  
  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```
Remind to [enable the new website configuration](#managing-active-website) and to [reload Apache configuration](#restarting-apache-service-to-load-new-configuration)