---
layout: post
title:  "Supporting A WordPress Site"
date:   2022-01-10 18:16:00 +0000
categories: wordpress mysql hosting
---
**WordPress** is one of the most popular platforms for developing sites; by their own estimate, "[over 43% of the web runs on WordPress][wordpress-feature-blurb]". It is massively configurable with a huge array of plugins to cover all sorts of features, from backups through to web commerce. It has a low barrier to entry, and tries hard to stay out of your way as much as possible so as to appeal to content creators who have very little technical expertise.

The flipside of this particular coin is that a free, massively popular, hugely configurable platform leads to an exponential amount of content about how best to support it, and a lot of the advice is irrelevant, inappropriate, or at worst utterly dangerous. Consider the following questions that I'm currently trying to answer, faced with a WordPress site that needs some updates, armed with nothing more than a login to the admin console (`wp_admin`) and one to the hosting provider:

- How do I take a copy of the site so I can roll it back if necessary?
- Does **WordPress** work well with source control? Can I store the relevant bits (see previous question) in a github repository?
- What about the data? What do I need from the **MySQL** installation? All databases? What about users?
- How do I run a **WordPress** site locally?
- If I make changes to my local site, how do I then get those changes into the production site? Is there a publish mechanism?
- What about changes to the data? How do I deal with a development **MySQL** database which is different from the production one? 

This post charts my five day journey to answer some of these questions. I start as a **WordPress** noob and end a **WordPress** noob, albeit one who managed to make a bit more sense of how to develop with WordPress (in a way that worked for me).

### So what makes this tick?

It seems like the bare minimum you need to run a WordPress site locally is a suitable LAMP stack (Linux, Apache, MySQL and PHP), so this seems like a good place to start. Installing these amounts to four incantations:

{% highlight bash %}
sudo apt-get install apache2
sudo apt-get install php5 libapache2-mod-php5
sudo apt-get install mysql-server
sudo apt-get install libapache2-mod-auth-mysql php5-mysql phpmyadmin
{% endhighlight %}

You can test these installations in various ways.

1. Open a browser and navigate to `http://localhost/` to verify your apache installation;
2. Create a file called `index.php` in a location from which apache will serve it (usually something like `/var/www`) and edit the file so it contains `<?php phpinfo(); ?>`. If you navigate to that page via a browser, you should see details on the PHP installation;
3. From a terminal, log in to sql using the username and password configured during installation - so something like `mysql -u root -p`;
4. For `phpMyAdmin` you should be able to navigate to `http://localhost/phpmyadmin`. The login details will be for the root user that you set up when configuring MySQL above;

> For subsequent SQL operations, you can use the MySQL console available via phpMyAdmin.

### Set up users

Using the root user everywhere is asking for all kinds of trouble, so you should configure a dedicated user for your WordPress site. To do this, let us first create a database in which the WordPress tables will live. Via phpMyAdmin this is as simple as selecting `New` from the left-hand database tree, and in the resulting panel, giving the database a name and choosing a collation strategy. (_Gotcha Number One_: the user that you entered to log in to phpMyAdmin needs to have the right permissions to create databases. If it doesn't you'll quickly find that there is no option to create a new database, and this caused me a fair bit of headscratching.)

> Collation may be important, so don't just select some random one. When choosing mine, I decided to look at the collation settings for the production database I was hoping to clone, so as to ensure they matched.

Having a database, I next created a user and granted that user full perms over the database. This might be overkill, but given the whole thing was local, I wasn't hugely paranoid about it. From the phpMyAdmin home, the _Users_ tab includes the option to create a new user; during creation you can choose what set of permissions to give it - including the option to limit access to the database you create in the previous step.

### Fill the database

I needed to ensure the content of the database matched that of the production site. **MySQL** provides a dump feature that can be used to emit the contents of a database to a file which can then be imported into another. 

I used the production server's phpMyAdmin to generate the dump of the relevant database, electing to save out a zipped result. This is then downloaded to your local machine. From the local machine's phpMyAdmin, select the correct database and choose the `Import` option - this option allows you to import directly from a zipped dump.

> _Gotcha Number Two_: The php server that powers the admin console is configured with a relatively low limit on the size of file it will allow to be uploaded. If your dump is large, then importing it is likely to fail unless this limit is addressed. For reference, in my case the default upload limit was 2MB, and the dumpfile I was trying to upload was 3.3MB. You can change this limit by updating the `upload_max_filesize` and `post_max_size` properties of the `php.ini` file that was created during the installation phase. If you are not sure where this file is located, you can look through the `phpinfo` output to find it. Don't forget you'll need to restart `apache` to pick this change up.

The upload took a few minutes to complete, but a clean status page at the end suggested it had all gone okay. Sure enough, in the database I had created, I could now see all sorts of WordPress tables, names starting with the tell-tale `wp_`as happens by default.

### Get a copy of the website files

This was a royal PITA. Depending on your hosting provider, you may have SFTP or SSH access to your host, allowing you to navigate and fetch the relevant files. In my case, I had to use some crappy web-based file navigation and download mechanism.

> _Aside: What files?_ Depending on where you look, the set of files you need to take a copy of varies from a handful (some references suggest the `wp-content` folder is enough, but I think you need the contents of the root folder down. As a rule of thumb, look for the `wp-config.php` file and assume that its containing folder is the one that is required (and all subfolders).

Once you have downloaded a copy of the content, you need to make some changes to make sure things work.

### Final Tweaks

1. Edit the `wp-config.php` file - this is in the root of the local copy of your website - to update the connection details for the SQL server. You are now connecting to the new database you created, on your local host, and using the username and password you set up for the purpose.
2. A few changes are required to the database content that you imported. These changes update values that tell WordPress where the site is located. The SQL should be relatively self-explanatory:

{% highlight sql %}
UPDATE wp_options SET option_value = replace(option_value, 'https://www.example.com', 'http://localhost/mylocalsite') WHERE option_name = 'home' OR option_name = 'siteurl';
UPDATE wp_posts SET post_content = replace(post_content, 'https://www.example.com', 'http://localhost/mylocalsite');
UPDATE wp_postmeta SET meta_value = replace(meta_value,'https://www.example.com','http://localhost/mylocalsite');
{% endhighlight %}

### A note about `mod_rewrite`

You might need to add a rewrite rule so that your local installation handles permalinks properly. WordPress provides nice links to pages that are suited for SEO (So pages have RESTful URLs like `http://foo.bar/category/fish` rather than `http://foo.bar/post.php?id=7`). In order for them to work, your Apache installation needs to have `mod_rewrite` installed and enabled, and you need to provide a rewrite rule that can be used to translate these links into the correct content.

On my Apache installation `mod_rewrite` was disabled by default. Enabling it is as simple as:

{% highlight bash %}
sudo a2enmod rewrite
{% endhighlight %}

The rewrite rule required for WordPress permalinks looks like this (this can be put in a file called `.htaccess` that sits alongside the rest of the site):

{% highlight bash %}
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /foo/
  RewriteRule ^index\.php$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /foo/index.php [L]
</IfModule>
{% endhighlight %}

You should also ensure that the apache directory configuration for your website allows overrides - otherwise the `.htaccess` file you painstakingly created above will be ignored. For example:

{% highlight bash %}
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>
{% endhighlight %}

Don't forget to restart apache to pick these changes up.

### Success?

At this point, you should find that navigating to the relevant address in your browser brings up a fully functioning copy of your website, but one which is now hosted locally.

[wordpress-feature-blurb]: https://wordpress.com/features/
