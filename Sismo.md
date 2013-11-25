How to set up the Sismo-CI server
====

### License stuff

This file is a part of the [tszym/wiki](https://github.com/tszym/wiki/) repository hosted on GitHub.

Copyright (c) 2013 Thomas Szymanski <contact@tszymanski.com>

For the full copyright and license information, please view the LICENSE file that was distributed with this document.

# The article

This article present the steps I followed to install the [Sismo](http://sismo.sensiolabs.org) continuous integration server on my Arch Linux PC. 
Sismo is provided by SensioLabs. Please visit the [GitHub](https://github.com/fabpot/Sismo).
With Sismo you can run a set of actions like PHPUnit tests automatically on each commit or other events. 
Sismo is used locally on each developer's machine.

Note that the instructions are mainly about projects made with [the Symfony 2 PHP framework](http://symfony.com/). Please adapt it for non-Symfony2 projects.

## Preconditions

You need to install [PHP](http://php.net) to run the CLI, and [Apache HTTP Server](https://httpd.apache.org) to run the GUI in a web-browser. 
Linux users can install the LAMP package. Please visit the [Arch Linux Wiki about LAMP](https://wiki.archlinux.org/index.php/Apache).

## Instructions (worked on Arch Linux)

### First steps

First of all, you have to download the single PHP file [here](http://sismo.sensiolabs.org/get/sismo.php).
As I wanted to use Sismo with CLI and GUI, I placed a directory called *sismo* in my *public_html* directory where I put all my web-projects. 
I think the equivalent on a WAMP server on Windows is the *www* directory.

For the rest of this article, this directory will be called *sismo directory* or *path to sismo*.

```
$ mkdir ~/public_html/sismo/
```

```
$ mv /where/was/sismo.php ~/public_html/sismo/
```

### Create the VirtualHost

I wanted to access the Sismo GUI by my web-browser at [http://sismo.local/sismo.php](http://sismo.local/sismo.php). 
For this, I needed to create a VirtualHost for my Apache HTTP server.

First, enable the custom VirtualHost file. So uncomment this line in the *httpd.conf* file.

```
# On my Arch distro, the file is in
# /etc/httpd/conf/httpd.conf
Include conf/extra/httpd-vhosts.conf # This is the line to uncomment
```

Then edit the file you included right before to add this!

```
# In /etc/httpd/conf/extra/httpd-vhosts.conf

<VirtualHost *:80>
    SetEnv SISMO_DATA_PATH "/path/to/sismo/data"
    SetEnv SISMO_CONFIG_PATH "/path/to/sismo/config.php"
    ServerAdmin yourmail@yourdomain.tld
    DocumentRoot "/path/to/sismo"
    ServerName sismo.local # The domain we put in the web-browser
    ErrorLog "/var/log/httpd/sismo-error_log"
    CustomLog "/var/log/httpd/sismo-access_log" common

    <Directory "/path/to/sismo">
        Order allow,deny
        Allow From All
    </Directory>
</VirtualHost>
```

Now, edit the */etc/hosts* file to make the *sismo.local* reachable.

```
# In /etc/hosts

127.0.0.1	sismo.localdomain	sismo.local	
```

To apply the changes, restart the Appache service. (May be reboot your machine to use the new host, I don't remember. Please contribute!)

### Add the path for the CLI

To use Sismo with the console, add this to your *~/.profile* or *~/.bashrc* :

```
export SISMO_DATA_PATH=/path/to/sismo/data/
export SISMO_CONFIG_PATH=/path/to/sismo/config.php
```

You will need to restart your console environment to use this. Just logout and login again.

### Some explainations about the sismo directory.

You certainly saw in the previous section about VirtualHost that we use a *data/* folder and a *config.php* file in the sismo directory. 
By default, Sismo will search in *~/.sismo/*. As I love having all about Sismo in only one Sismo folder, I put the previous elements in the sismo directory. 
The *config.php* file describe the projects watched by Sismo. To do his job, Sismo dosen,'t work in your repository. He will clone it in his *data/* folder.

By the way, Sismo need to acces and write in his folder!

### Giving access to the data folder

To run my web projects, I put my user and *httpd* in a group called *httpworkers*. Then all I have to do is giving the access to this group.
In the sismo directory, run the following: 

```
$ sudo setfacl -R -m g:httpworkers:rwX .
$ sudo setfacl -dR -m g:httpworkers:rwx .
```

Make sure the *sismo.php* is readable by everyone: 

```
$ chmod +r sismo.php
```

### The config.php file

Here is an example inspired of the [GitHub of Sismo](https://github.com/fabpot/Sismo) for the *config.php* file. 
It shows you the changes I made and use. 
Please modify as you need.

```php

<?php

$projects = array();

// create a Growl notifier (for MacOS X)
//$notifier = new Sismo\Notifier\GrowlNotifier('pa$$word');

// create a DBus notifier (for Linux)
$notifier = new Sismo\Notifier\DBusNotifier();

//Set default command to run (use composer if the project has to use it)
Sismo\Project::setDefaultCommand('if [ -f composer.json ]; then composer install --no-interaction --dev; fi && phpunit -c app/ src/');

//-----------------
// Projects section
//-----------------

// To use an awesome script later in this article, please follow
// this simple name-convention for your project
// (the first parameter of the objects)
// $name = "my-project-branchname";

// add a local repository hosted on Github
$projects[] = new Sismo\GithubProject('Twig-Local', '/Users/fabien/Twig', $notifier);

// add a remote Github repository
$projects[] = new Sismo\GithubProject('Twig', 'fabpot/Twig', $notifier);

// Add a project with two branches to check.
// Just give differents names (first parameter) 
// and the @branch in the path
$projects[] = new Sismo\GithubProject('tszym-myproject-master', '/path/to/repo/myproject@master', $notifier);
$projects[] = new Sismo\GithubProject('tszym-myproject-develop', '/path/to/repo/myproject@develop', $notifier);

// add a project with custom settings
$sf2 = new Sismo\Project('Symfony');
$sf2->setRepository('https://github.com/symfony/symfony.git');
$sf2->setBranch('master');
$sf2->setCommand('./vendors.sh; phpunit');
$sf2->setSlug('symfony-local');
$sf2->setUrlPattern('https://github.com/symfony/symfony/commit/%commit%');
$sf2->addNotifier($notifier);
$projects[] = $sf2;

return $projects;
```

### Build with Sismo

Now you're ready to run a build.

```
$ php sismo.php build --verbose
```

You will certainly see some notifications about the build in nour notification zone (my first one was a fail notification :( ).
Now visit your [sismo.local/sismo.php](http://sismo.local/sismo.php) page in your web-browser and be happy to see the results!

### Auto build on commit hook

A build can be triggered as a git hool. 
Just create the file in *repo/.git/hooks/* and add the following bash script. 

You were invited to follow name-conventions to make this script run on the commits of each branch you put an entry on in the *config.php* file. 
For example, I put the branches *master* and *develop* in the *config.php* file. 
Every commit in branches *master* and *develop* will run a build for the current branch. 
Commits in the branch *myfeature* will not run a build since I didn't put this branch in the *config.php* file. 

Please modify the *PROJECT_NAME* and the *path to sismo* only.

```bash

#!/bin/bash

# Get the name of the current branch
BRANCH_NAME=`git rev-parse --abbrev-ref HEAD`

# Specify the project name as given in the Sismo config.php file
# Do not put here the branch name
#
# Example:
# In config.php 'tszym-myproject-master' for the master branch of the project
# PROJECT_NAME="tszym-myproject"

nohup php /path/to/sismo/sismo.php --quiet --force build    $PROJECT_NAME-$BRANCH_NAME `git log -1 HEAD --pretty="%H"` &>/dev/null &

```

Sismo will build automatically after each commit on the repository.
Don't forget to set the file executable with *chmod +x post-commit*.

## Knowns errors

### Something went wrong on the web-page

Make sure Sismo can write in the *data/* folder. When running a build, you should see files in the *data/* folder.

### I got an exception about SQLite3 when running the CLI

Make sure the sqlite lib is installed on your system and enabled in Appache. 
With Arch Linux, I needed the *php-sqlite* package.

Then enable the extension in your *php.ini* file:

```
extension=sqlite3.so
```

You need to restart your http service now.

### I have a problem not documented here

Please search the solution on the web, try it and contribute here to help the following developer who will get the same problem.

