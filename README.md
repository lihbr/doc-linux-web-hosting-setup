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
    1. [Moving default directory](#moving-default-directory)
    2. [Creating subdomain site](#creating-subdomain-site)
6. [MySQL configuration](#mysql-configuration)
    1. [Define root password](#define-root-password)

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

## Moving default directory

We'll move default directory (`/var/www`) to `/var/web/www` as `/var/web` will be our root folder for every subdomain.

First we'll need to edit some apache default configuration file:
```
$ vi /etc/apache2/sites-enabled/000-default.conf
```

Update it to something like this:
```apache
<VirtualHost *:80>
  ServerAdmin webmaster@localhost
  DocumentRoot /var/web/www
  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Then we'll do the edit the ssl configuration file:
```
$ vi /etc/apache2/sites-available/ssl-default.conf
```

Then to update access permission you can edit the end of apache2.conf:
```
$ vi /etc/apache2/apache2.conf
```
A basic implementation could be:
```apache
<Directory />
  Options FollowSymLinks
  AllowOverride None
  Require all denied
</Directory>

<Directory /var/web/*/>
  Options Indexes FollowSymLinks MultiViews
  AllowOverride All
  Require all granted
</Directory>
```

Just update the `DocumentRoot` property:
```apache
DocumentRoot /var/web/www
```

To check if your new configuration is correct you can prompt the following:
```
$ apachectl configtest
```

If everything is ok, then you'll need to restart Apache:
```
$ service apache2 restart
```

## Creating subdomain site

First create a directory in `/var/web` using (once you're at the right location ofc):
```
$ mkdir DIRECTORY_NAME
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