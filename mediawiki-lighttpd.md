# How to Install MediaWiki and an LLMP (Linux, Lighttpd, MySQL, and PHP) Stack on Ubuntu 14.04

### Introduction

MediaWiki is a popular open source wiki platform that can be used for public or internal collaborative content publishing.  MediaWiki is used for many of the most popular wikis on the internet including Wikipedia, the site that the project was originally designed to serve.

In this guide, we will be setting up the latest version of MediaWiki on an Ubuntu 14.04 server.  We will use the `lighttpd` web server to make the actual content available, `php-fpm` to handle dynamic processing, and `mysql` to store our wiki's data.

## Prerequisites

To complete this guide, you should have access to a clean Ubuntu 14.04 server instance.  On this system, you should have a non-root user configured with `sudo` privileges for administrative tasks.  You can learn how to set this up by following our [Ubuntu 14.04 initial server setup guide](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04).

When you are ready to continue, log into your server with your `sudo` user and get started below.


## Goals

At the end of this tutorial you will have MediaWiki running on `lighttpd` on a Ubuntu 14.04 server.

We'll install and test each component as we go. There are five main sections to this tutorial:

* Installing the `lighttpd` web server
* Installing PHP and integrating it with `lighttpd`
* Installing MySQL and setting up a database for MediaWiki
* Installing MediaWiki
* Tidying Up

We'll be using `apt` to install the first three components (`lighttpd`, PHP, and MySQL) from Ubuntu's package repositories. We'll get MediaWiki directly from [MediaWiki.org](https://www.mediawiki.org/wiki/MediaWiki) to make sure that we have the latest version.


## Step 1 -- Installing lighttpd

[Lighttpd](http://www.lighttpd.net/) is an open source web server that focuses on increased performance and a light memory footprint. Together with the very popular MySQL database server and the PHP server side dynamic scripting language, Lighttpd is a strong alternative to the more resource intensive
Apache server.


Since this is our first time using `apt` on this server, we start off by updating our local package index. We can then install `lighttpd`. The `-y` option assumes a "yes" answer to any prompts.

```command
sudo apt-get -y update
sudo apt-get -y install lighttpd
```

Lighttpd is configured to start up after installation. To make sure it's running, open a browser and type in your Ubuntu server's domain name or public IP address:

```
http:<^>your_server_ip<^>
```

You should see the default `lighttpd` "placeholder page".

![lighttpd placeholder page](http://i.imgur.com/E6V2Rwd.png)

The content that `lighttpd` serves is located at `/var/www/`. This is where we'll install MediaWiki later.


## Step 2 -- Installing PHP

Now that `lighttpd` is working, the next step is to install PHP, the language that MediaWiki uses.

Lighttpd needs extra software to run PHP code. For this installation, we'll use `php5-fpm`, a process manager for PHP. There are other PHP process managers, such as `php5-cgi`. We're using `php5-fpm` because it runs PHP in a separate persistent process and uses less memory.


To install PHP, we'll install the modules `php5-fpm` and `php5-mysql`. Installing these two modules automatically installs the basic PHP package as well.

```command
sudo apt-get -y install php5-fpm php5-mysql
```

### Setting Up php5-fpm

The next step is to configure `lighttpd` to work with `php5-fpm`. It's important to follow these next steps in order.

Go to the `lighttpd` configuration directory `/etc/lighttpd/conf-available`. This directory contains all of the *available* `lighttpd` modules. None of these modules has been enabled yet.

```
cd /etc/lighttpd/conf-available
```

You're going to make a change to the file `15-fastcgi-php.conf` in this directory. By default, this configuration is set up to work with `php5-cgi`. We're going to change it so that it uses `php5-fpm` instead.

Make a backup of the file before you change it:

```command
sudo cp 15-fastcgi-php.conf 15-fastcgi-php.conf.original
```

Now open `15-fastcgi-php.conf` in an editor. Remember to use `sudo` so you can edit the file.

```command
sudo nano 15-fastcgi-php.conf
```

Find the line that begins with `fastcgi.server += ( ".php" =>`. Replace the lines between `((` and `))`` so that this section of the file looks like this:

```
fastcgi.server += ( ".php" =>
        ((
                "socket" => "/var/run/php5-fpm.sock",
                "broken-scriptfilename" => "enable"
        ))
)
```

Save the file and exit the editor (`CTRL+X`, then `Y`).

Next, enable the CGI modules for `lighttpd`.

```command
sudo lighttpd-enable-mod fastcgi
sudo lighttpd-enable-mod fastcgi-php
```

These commands make symbolic links to the configuration files in the `/etc/lighttpd/conf-available`.

To activate these changes, force `lighttpd` to reload its configuration files.

```command
sudo service lighttpd force-reload
```

Check that `lighttpd` restarted successfully.

```command
sudo service lighttpd status
```

The result should be `lighttpd is running`. If it's not, check your edits to `15-fastcgi-php.conf`.

### Checking That php5-fpm Works

To make sure that everything works, you'll insert a small PHP program that runs `phpinfo()` in `/var/www`, the directory that holds the files that `lighttpd` serves.

Create the file in an editor.

```command
sudo nano /var/www/phpinfo.php
```

Paste this line of PHP code in it:

```command
<?php phpinfo(); ?>
```

Now point your browser to that file.

```
http:<^>your_server_ip<^>/phpinfo.php
```

You should see server information page. The entry labeled **Server API** should say `FPM/FastCGI`.

![phpinfo](http://i.imgur.com/ZHXYW55.png)


### Installing Additional PHP Modules for MediaWiki

Before we finish up with PHP installation, let's install some modules that MediaWiki uses. All of these are optional, but the first two are particularly recommended.

`php5-intl` provides internationalization support.

```command
sudo apt-get -y install php5-intl
```

`php5-gd` provides support for creating image thumbnails.

```command
sudo apt-get -y install php5-gd
```

`php5-xcache` caches pages so they don't have to be rerendered if there are no changes. Be sure to restart `php5-fpm` so it knows that you're using `php5-xcache`.

```commmand
sudo apt-get -y install php5-xcache
sudo service php5-fpm restart
```

If you will be displaying mathematical equations, you will want to install `texlive`. This package has a lot of dependencies, so if you don't need it, you may not want to install it. You can always install it later.

```command
sudo apt-get -y install texlive
```

Reload `lighttpd` to pick up the new modules.

```command
sudo service lighttpd force-reload
```

Check that `lighttpd` restarted successfully.

```command
sudo service lighttpd status
```

PHP is now installed and integrated with `lighttpd`. Setting up the database is next.

## Step 3 -- Installing MySQL

In this step, you'll install the MySQL database package. MediaWiki uses a database to store its contents and to manage wiki users.

When you install MySQL, you'll be asked to make up a password for the `root` user. This `root` user is the main MySQL account, not the Linux `root` user you used to log in to your Droplet the first time.

The MySQL `root` user has complete access to all of the databases in your MySQL server. While it's possible to use the `root` user to manage MediaWiki's database, we'll create a separate database user.

```command
sudo apt-get -y install mysql-server
```

The installation will pause for you to enter a password for the `root` user.

### Preparing and Securing MySQL

When the package installation finishes, there are a couple more steps to make MySQL ready to use.

First, set up the directories where MySQL stores its database files.

```command
sudo mysql_install_db
```

Next, clean up and secure MySQL. The `mysql_secure_installation` script asks you several questions. Since you already set up a `root` password when you installed MySQL, you can answer `no` for that question and answer `yes` (the default) for all the other questions.

```command
sudo mysql_secure_installation
```
### Creating the MediaWiki Database

With the basic MySQL software installed, we can create the database that MediaWiki will use.

First log in to MySQL:

```command
mysql -u root -p
```

Enter the `root` password you created when you installed MySQL. The prompt will change to `mysql>` to indicate that you'll be entering MySQL commands.

Create the database the MediaWiki will use. You can use any name you like. For this tutorial, we'll name the database `wikidb`.

```
CREATE DATABASE <^>wikidb<^>;
```

Don't forget to end MySQL commands with a semicolon (`;`).

Next, create a database user to work with the `wikidb` database.  This is a long command, so we'll split it into three lines. The prompt will change to `->` to indicate that you haven't finished entering the command. The semicolon at the end completes the command.

```
GRANT INDEX, CREATE, SELECT, INSERT, UPDATE,
DELETE, ALTER, LOCK TABLES ON
<^>wikidb<^>.* TO '<^>sammy<^>'@'localhost' IDENTIFIED BY '<^>password<^>';
```

If you chose a different name for your database, use it instead of `<^>wikidb<^>`. You can use any name and password for the database user `<^>sammy<^>` and for `<^>password<^>`. Be sure to remember these three values. You will need them when you install MediaWiki.

Finally, make sure that all the information you entered for the database user is stored properly:

```
FLUSH PRIVILEGES;
```

And exit the MySQL command prompt. The `exit` command does not need a semicolon.

```
exit
```

At this point you've installed the web server `lighttpd`, PHP, and MySQL. You now have a full LLMP (Linux, Lighttpd, MySQL, PHP) stack.

## Step 4 -- Installing MediaWiki

With your LLMP stack set up, you're ready to install MediaWiki.

Although there is a MediaWiki `apt` package, it is not the latest version. Also, MediaWiki recommends that you install from their website.

### Downloading MediaWiki

We'll download the MediaWiki tarball to our home directory and unpack it there. Then we'll move it into place in the `lighttpd` web directory `/var/www`.

Make sure you're at your root directory:

```command
cd ~
```

Use `curl` to download the tarball to your Droplet.

```command
curl -O http://releases.wikimedia.org/mediawiki/1.25/mediawiki-1.25.1.tar.gz
```

The latest stable version of MediaWiki is on their [Download](https://www.mediawiki.org/wiki/Download) page. If the version listed there is different from the one in the `curl` command above, right-click (CTRL+Click on a Mac) on the **Download MediaWiki** link on that page to copy the link and use that link instead of the one above.

Unpack the tarball with the `tar` command. A bunch of text will fly by on your screen.

```command
tar xvzf mediawiki-1.25.1.tar.gz
```

If use the `ls` command in your directory, you'll see a directory named `mediawiki-1.25.1` and the tarball, `mediawiki-1.25.1.tar.gz`.

### Moving MediaWiki Files into Place

Before we move MediaWiki into `lighttpd`'s document root, let's clean out the placeholder page and the `phpinfo.php` we put there when we set up PHP:

```command
sudo rm /var/www/index.lighttpd.html /var/www/phpinfo.php
```

To move the contents of the `mediawiki-1.25.1` directory to `lighttpd`'s document root, run this command:

```command
sudo mv mediawiki-1.25.1/* /var/www
```

### Setting Up MediaWiki

Now you're ready to use MediaWiki's setup process. Point your browser to your Droplet:

```
http:<^>your_server_ip<^>
```

Click the **set up the wiki** link to start.

The first page lets you choose the language the installer should use during installation and the main language of the wiki itself. The default for both is English. Click **Continue**.

The next page checks whether MediaWiki can run on your machine. Look for green text that says "The environment has been checked. You can install MediaWiki". Click **Continue**.

In the **Connect to database** page, you'll enter information about your database. You'll need the name of the database you created when you installed MySQL, the database user, and the database user's password.


![connect to database](http://i.imgur.com/SMuYIcu.png)


After entering the information about your database, click **Continue**.

In the **Database settings** page, leave the defaults selected, and click **Continue**.

In the **Name** page, give your wiki a name. In the administrator account, set up and account for the wiki administrator.

You can select **I'm bored already, just install the wiki**, but if you want to enable some options such as file uploading or caching, select **Ask me more questions**, and click **Continue**.

Each section in the **Options** page has help text that describes what the options do. If you installed `php5-xcache`, select **PHP object caching (APC, XCache or WinCache)** to enable caching.

Click **Continue** to finish setting up the options.

In the **Install** page you have an opportunity to go back to change any options you selected. When you're ready, click **Continue** to finish the installation.

When you reach the **Complete** page, you'll see this:

![setup complete](http://i.imgur.com/mdkfXpu.png)


A file named `LocalSettings.php` should start downloading automatically. You need to move this file to the `lighttpd` document root, so make sure to download the file before closing the page.

To move `LocalSettings.php` from your local machine to your Droplet, open it in an editor on your local computer, and copy the contents. Then use an editor to create the file on the server:

```commands
sudo nano /var/www/LocalSettings.php
```

Paste the contents that you copied into this new file. Save it and exit.

Back in your browser, click the **enter your wiki** link, and your wiki is ready to use.


## Step 5 -- Tidying Up

Just one more thing to do. By default, the `/var/www` directory is owned by `root`, and if you followed the steps for downloading and moving the MediaWiki files, those files are owned by your `sudo` user.

To improve security, the `/var/www` directory should be owned by the `www-data` user, which is the account that runs `lighttpd`. To do this, use the `chown` (change owner) command:

```command
sudo chown -R www-data:www-data /var/www
```

## Next Steps

You now have MediaWiki running on a LLMP stack.

To learn more about managing a MediaWiki, see the [MediaWiki System Administration Manual](https://www.mediawiki.org/wiki/Manual:System_administration).

You will probably want to replace the default logo that tells you to change the logo. A quick way to do that is to copy the MediaWiki logo to the `images` directory:

```command
sudo cp /var/www/resources/assets/mediawiki.png /var/www/images/
```

Then edit `/var/www/LocalSettings.php`. Find the line that begins with `$wgLogo`, and replace it with this:

```
$wgLogo = "$wgResourceBasePath/resources/assets/mediawiki.png";
```

When you associate your Droplet with a [domain name](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-host-name-with-digitalocean), you will want to edit `/var/www/LocalSettings.php`. Find the line that begins with `$wgServer` and replace the URL there with your domain.

```
## The protocol and server name to use in fully-qualified URLs
$wgServer = "http://<^>example.com<^>";
```





