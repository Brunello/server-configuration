Add New Site to Server
======================

Create a new configuration file:

    $ sudo vi /etc/apache2/sites-available/<domainname.com>

> File Contents:

    <VirtualHost *:80>
        ServerName www.domainname.com
        Redirect permanent / http://domainname.com/
    </VirtualHost>

    <VirtualHost <serveripaddress>:80>
         ServerAdmin noreply@domainname.com
         ServerName domainname.com
         DocumentRoot /var/www/vhosts/domainname.com/public_html/
         ErrorLog /var/logs/domainname.com/logs/error.log
         CustomLog /var/logs/domainname.com/logs/access.log combined
    </VirtualHost>

> Adjust the above to your needs. Assuming the site is not on a subdomain and
> that you want to redirect traffic from www.domainname.com to domainnam.com.

Create the site directories

    $ sudo mkdir -p /var/www/vhosts/domainname.com/public_html
    $ sudo mkdir -p /var/logs/domainname.com/logs

Create a new user that will own these site:

    $ sudo useradd <username>

> Use the following pattern when choosing a username:

    <domainname before TLD>-<site instance>

> For example: `domainname-prod` or `domainname-staging`

Set a password for the new account

    $ sudo passwd <username>

Change the user's home directory to the site's webroot

    $ sudo usermod -d /var/www/vhosts/domainname.com/ <username>

Enable the site

    $ sudo a2ensite domainname.com

Reload apache

    $ sudo service apache2 reload

Change the group and owner of the site's docroot to the site owner account

    $ sudo chown -R <username>:www-data /var/www/vhosts/domainname.com
