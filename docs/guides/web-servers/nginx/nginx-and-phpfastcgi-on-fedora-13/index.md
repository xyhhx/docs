---
slug: nginx-and-phpfastcgi-on-fedora-13
author:
  name: Linode
  email: docs@linode.com
description: 'Serve dynamic websites and applications with the lightweight nginx web server and PHP-FastCGI on Fedora 13'
keywords: ["nginx", "nginx fedora 13", "nginx fastcgi", "nginx php"]
tags: ["web server","fedora","php","nginx"]
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
aliases: ['/websites/nginx/nginx-and-phpfastcgi-on-fedora-13/','/web-servers/nginx/php-fastcgi/fedora-13/','/web-servers/nginx/nginx-and-phpfastcgi-on-fedora-13/']
modified: 2011-05-17
modified_by:
  name: Linode
published: 2010-05-27
title: 'Nginx and PHP-FastCGI on Fedora 13'
deprecated: true
relations:
    platform:
        key: nginx-php-fastcgi
        keywords:
            - distribution: Fedora 13
---

The nginx web server is a fast, lightweight server designed to efficiently handle the needs of both low and high traffic websites. Although commonly used to serve static content, it's quite capable of handling dynamic pages as well. This guide will help you get nginx up and running with PHP and FastCGI on your Fedora 13 Linode.

It is assumed that you've already followed the steps outlined in our [Setting Up and Securing a Compute Instance](/docs/guides/set-up-and-secure/). These steps should be performed via a root login to your Linode over SSH.

## Basic System Configuration

Issue the following commands to set your system hostname, substituting a unique value for "hostname." :

    echo "HOSTNAME=hostname" >> /etc/sysconfig/network
    hostname "hostname"

Edit your `/etc/hosts` file to resemble the following, substituting your Linode's public IP address for 12.34.56.78, your hostname for "hostname," and your primary domain name for "example.com." :

{{< file "/etc/hosts" >}}
127.0.0.1 localhost.localdomain localhost
12.34.56.78 hostname.example.com hostname

{{< /file >}}


## Install Required Packages

Issue the following commands to update your system and install the nginx web server, PHP, and compiler tools:

    yum update
    yum install nginx php-cli php make automake gcc gcc-c++ spawn-fcgi wget
    chkconfig --add nginx
    chkconfig --level 35 nginx on
    service nginx start

Once the installation process finishes, you may wish to make sure nginx is running by browsing to your Linode's IP address (found on the **Networking** tab in the [Linode Cloud Manager](http://cloud.linode.com/)). You should get the default NGINX page.

## Configure Your Site

In this guide, we'll be using the domain "example.com" as our example site. You should substitute your own domain name in the configuration steps that follow. First, we'll need to create directories to hold our content and log files:

    mkdir -p /srv/www/www.example.com/public_html
    mkdir /srv/www/www.example.com/logs
    chown -R nginx:nginx /srv/www/www.example.com

Issue the following commands to create virtual hosting directories:

    mkdir /etc/nginx/sites-available
    mkdir /etc/nginx/sites-enabled

Add the following lines to your `/etc/nginx/nginx.conf` file, immediately after the line for `include /etc/nginx/conf.d/*.conf`:

{{< file "/etc/nginx/nginx.conf" nginx >}}
# Load virtual host configuration files.
include /etc/nginx/sites-enabled/*;

{{< /file >}}


Next, define your site's virtual host file:

{{< file "/etc/nginx/sites-available/www.example.com" nginx >}}
server {
    server_name www.example.com example.com;
    access_log /srv/www/example.com/www/logs/access.log;
    error_log /srv/www/example.com/www/logs/error.log;
    root /srv/www/example.com/www/public_html;

    location / {
        index index.html index.htm index.php;
    }

    location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_pass  127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME /srv/www/example.com/www/public_html$fastcgi_script_name;
    }
}

{{< /file >}}


**Important security note:** If you're planning to run applications that support file uploads (images, for example), the above configuration may expose you to a security risk by allowing arbitrary code execution. The short explanation for this behavior is that a properly crafted URI which ends in ".php", in combination with a malicious image file that actually contains valid PHP, can result in the image being processed as PHP. For more information on the specifics of this behavior, you may wish to review the information provided on [Neal Poole's blog](https://nealpoole.com/blog/2011/04/setting-up-php-fastcgi-and-nginx-dont-trust-the-tutorials-check-your-configuration/).

To mitigate this issue, you may wish to modify your configuration to include a `try_files` directive. Please note that this fix requires nginx and the php-fcgi workers to reside on the same server.

{{< file "/etc/nginx/sites-available/www.example.com" nginx >}}
location ~ \.php$ {
    try_files $uri =404;
    include /etc/nginx/fastcgi_params;
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME /srv/www/example.com/www/public_html$fastcgi_script_name;
}
{{< /file >}}

Additionally, it's a good idea to secure any upload directories your applications may use. The following configuration excerpt demonstrates securing an "/images" directory.

{{< file "/etc/nginx/sites-available/www.example.com" nginx >}}
location ~ \.php$ {
    include /etc/nginx/fastcgi_params;
    if ($uri !~ "^/images/") {
        fastcgi_pass 127.0.0.1:9000;
    }
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME /srv/www/example.com/www/public_html$fastcgi_script_name;
}
{{< /file >}}

After reviewing your configuration for potential security issues, issue the following commands to enable the site:

    cd /etc/nginx/sites-enabled/
    ln -s /etc/nginx/sites-available/www.example.com
    service nginx restart

You may wish to create a test HTML page under `/srv/www/www.example.com/public_html/` and view it in your browser to verify that nginx is properly serving your site (PHP will not work yet). Please note that this will require an [entry in DNS](/docs/products/networking/dns-manager/guides/common-dns-configurations/) pointing your domain name to your Linode's IP address.

## Configure spawn-fcgi

Issue the following command sequence to download scripts to control spawn-fcgi and php-fastcgi, set privileges, make the init script run at startup, and launch it for the first time:

    cd /opt
    wget -O php-fastcgi-rpm.sh http://www.linode.com/docs/assets/647-php-fastcgi-rpm.sh
    mv php-fastcgi-rpm.sh /usr/bin/php-fastcgi
    chmod +x /usr/bin/php-fastcgi
    wget -O php-fastcgi-init-rpm.sh http://www.linode.com/docs/assets/648-php-fastcgi-init-rpm.sh
    mv php-fastcgi-init-rpm.sh /etc/rc.d/init.d/php-fastcgi
    chmod +x /etc/rc.d/init.d/php-fastcgi
    chkconfig --add php-fastcgi
    chkconfig php-fastcgi on
    /etc/init.d/php-fastcgi start

## Test PHP with FastCGI

Create a file called "test.php" in your site's "public\_html" directory with the following contents:

{{< file "/srv/www/www.example.com/public\\_html/test.php" php >}}
<?php echo phpinfo(); ?>

{{< /file >}}


When you visit `http://www.example.com/test.php` in your browser, the standard "PHP info" output is shown. Congratulations, you've configured the nginx web server to use PHP-FastCGI for dynamic content!

## More Information

You may wish to consult the following resources for additional information on this topic. While these are provided in the hope that they will be useful, please note that we cannot vouch for the accuracy or timeliness of externally hosted materials.

- [The NGINX Homepage](http://nginx.org/)
- [FastCGI Project Homepage](http://www.fastcgi.com/)
- [PHP Documentation](http://www.php.net/docs.php)
- [Installing NGINX on Fedora 13](/docs/guides/websites-with-nginx-on-fedora-13/)
- [Basic NGINX Configuration](/docs/guides/how-to-configure-nginx/)
