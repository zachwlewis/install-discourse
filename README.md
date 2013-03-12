# Installing Discourse on Ubuntu and EC2
Original copyright 2013 by Christopher Baus <christopher@baus.net>. Licensed under GPL 2.0

Updated version copyright 2013 by Lee Dohm <lee@liftedstudios.com>

Discourse is [web discussion forum software](http://discourse.org) by Jeff Atwood (et al.). Considering the state of forum software, and Jeff's previous success with StackOverflow, I'm confident it is going to be a success. With that said it is still in a very early state, and if you are not an expert on Linux and Ruby on Rails administration, getting a Discourse site up and running can be a daunting task. 

Hopefully the document will be useful for someone who has some Linux administration experience and wants to run and administrate their own Discourse server. I am erring on the side of verbosity.

## Create EC2 Instance with Ubuntu 12.04 LTS x64

While these instructions should work fine on most Ubuntu installations, I have explicitly tested them on Amazon EC2.

I decided on Ubuntu 12.04 LTS x64 since it is the flavor of Ubuntu that the [main Discourse installation](http://meta.discourse.org) is run upon.

Before creating your EC2 instance, you should register the domain name you want to use for your forum. I'm using discoursetest.org for this document, and forum.discoursetest.org as the FQDN.  

After creating your account at Amazon AWS, launch an instance *with at least 1GB of RAM* [1], and select the Ubuntu OS image you want to use. I set the Hostname to forum.discoursetest.org. 

You will need to allocate an Elastic IP address and associate it with your new EC2 instance after you've started it.  You should go to your domain registrar and set the DNS records to point to your new IP.  I've set both the * and @ records to point to the instance's IP. This allows the root domain and all sub-domains to resolve to instance's IP address. 

[1] A minimum of 1GB of RAM is required to compile assets for production.  At the time of this writing, an `m1.small` instance is the smallest instance that has 1GB of RAM.

## Log in to Your Server

I will use discoursetest.org when a domain name is required in the installation. You should replace discoursetest.org with your own domain name. If you are using OS X or Linux, start a terminal and ssh to your new server. Windows users should consider installing [Putty](http://putty.org/) to access your new server.

## Create a User Account

While the Amazon EC2 Ubuntu images all have a non-root `ubuntu` admin user, I chose to create a more personal admin user account. For the purposes of this document, I'm going to call the new user `admin`.

Adding the user to the sudo group will allow the user to perform tasks as root using the [sudo](https://help.ubuntu.com/community/RootSudo) command. 

```bash
$ sudo adduser admin
$ sudo adduser admin sudo
```
## Log in Using the Admin Account

```bash
$ logout
# now back at the local terminal prompt
$ ssh admin@discoursetest.org
```

## Use `apt-get` to Install Core System Dependencies

The apt-get command is used to add packages to Ubuntu (and all Debian based Linux distributions). The Amazon EC2 Ubuntu images come with a limited configuration, so you will have to install many of the software the dependencies yourself.

To install system packages, you must have root privledges. Since the admin account is part of the sudo group, the admin account can run commands with root privledges by using the sudo command. Just prepend sudo to any commands you want to run as root. This includes apt-get commands to install packages.

```bash
# Install required packages
$ sudo apt-get install build-essential postgresql-9.1 postgresql-contrib-9.1 libxml2-dev libxslt-dev libpq-dev redis-server nginx postfix
```

During the installation, you will be prompted for Postfix configuration information. [Postfix](https://help.ubuntu.com/community/Postfix) is used to send mail from Discourse. Just keep the default "Internet Site."

At the next prompt just enter your domain name. In my test case this is discoursetest.org.

## Edit Configuration Files

At various points in the installation procedure, you will need to edit configuration files with a text editor. `vi` is installed by default and is the de facto standard editor used by admins (although it appears that `nano` is configured as the default editor for many uses in the Ubuntu image), so I use `vi` for any editing commands, but you may want to consider installing the editor of your choice.

## Set the Host Name

EC2's provisioning procedure doesn't assume your instance will require a hostname when it is created. I'd recommend editing /etc/hosts to correctly contain your hostname.

```bash
$ vi /etc/hosts
```
The first line of my /etc/hosts file looks like:

```bash
127.0.0.1  forum.discoursetest.org forum localhost
```

You should replace discoursetest.org with your own domain name. 

## Configure Postgres User Account

Discourse uses the Postgres database to store forum data. The configuration procedure is similar to MySQL, but I am a Postgres newbie, so if you have improvements to this aspect of the installation procedure, please let me know.

Note: this is the easiest way to setup the Postgres server, but it also creates a highly privledged Postgres user account. Future revisions of this document may offer alternatives for creating the Postgres DBs, which would allow Discourse to login to Postgres as a user with lower privledges.

```bash
$ sudo -u postgres createuser admin -s -P
```

It will ask for a password for the account, so pick one and remember it.  You will need the password for this database account when editing the Discourse database configuration file.

## Install and Configure RVM and Ruby

I chose to use RVM to manage installations of Ruby and Gems.  These instructions will employ a multi-user RVM installation so that all user accounts will have access to RVM, Ruby and the gemsets we will create.

First, install RVM and add your admin account to the `rvm` group:

```bash
$ \curl -L https://get.rvm.io | sudo bash -s stable
$ sudo adduser admin rvm
```

After adding yourself to the `rvm` group, you will need to log out and log back in to register the change and activate RVM for your session.

Then install Ruby v1.9.3 and create a gemset for Discourse:

```bash
$ rvm install 1.9.3
$ rvm use --default 1.9.3
$ rvm gemset create discourse
```

*Note*: Some people are using Ruby v2.0 for their installations to good effect, but I have not tested version 2 with these instructions.

## Pull and Configure the Discourse Application

Now we are ready install the actual Discourse application. This will pull a copy of the Discourse app from my own branch. The advantage of using this branch is that it has been tested with these instructions, but it may fall behind the master which is rapidly changing. 

```bash
# Pull the latest version from github.
$ git clone https://github.com/lee-dohm/discourse.git
$ cd discourse
```

Create an `.rvmrc` file for Discourse:

```bash
$ echo "rvm 1.9.3@discourse" > .rvmrc
```

## Set Discourse Application Settings

Now you have set the Discourse application settings. The configuration files are in a directory called `config`.  There are sample configuration files now included, so you need to copy these files and modify them with your own changes.

```
$ cd ~/discourse/config
$ cp ./database.yml.sample ./database.yml
$ cp ./redis.yml.sample ./redis.yml
```

Now you need to edit the configuration files and apply your own settings. 


Start by editing the database configuration file which should be now located at `~/discourse/config/database.yml`.

```bash
$ vi ~/discourse/config/database.yml
```

Edit the file to add your Postgres username and password to each configuration in the file. Also add `localhost` to the production configuration because the production DB will also be run on the localhost in this configuration.

When you are done the file should look similar to:

```yaml
development:
  adapter: postgresql
  database: discourse_development
  username: admin
  password: <your_postgres_password>
  host: localhost
  pool: 5
  timeout: 5000
  host_names:
    - "localhost"

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  adapter: postgresql
  database: discourse_test
  username: admin
  password: <your_postgres_password>
  host: localhost
  pool: 5
  timeout: 5000
  host_names:
    - test.localhost

# using the test db, so jenkins can run this config
# we need it to be in production so it minifies assets
production:
  adapter: postgresql
  database: discourse
  username: admin
  password: <your_postgres_password>
  host: localhost
  pool: 5
  timeout: 5000
  host_names:
    - production.localhost
```

I'm not a fan of entering the DB password as clear text in the database.yml file. If you have a better solution to this, let me know. 

## Deploy the Database and Start the Server

Now you should be ready to deploy the database and start the server.

This will start the development environment on port 3000.

```bash
$ cd ~/discourse
# Set Rails configuration
$ export RAILS_ENV=development
$ rake db:create
$ rake db:migrate
$ rake db:seed_fu
$ thin start
```

I tested the configuration by going to http://discoursetest.org:3000/

## Installing the Production Environment

**WARNING: very preliminary instructions follows**

### Setup the `www-data` Account

```bash
$ sudo mkdir /var/www
$ sudo chgrp www-data /var/www
$ sudo chmod g+w /var/www
```

### Configure nginx

```bash
$ cd ~/discourse/
$ sudo cp config/nginx.sample.conf /etc/nginx/sites-available/discourse.conf
$ sudo ln -s /etc/nginx/sites-available/discourse.conf /etc/nginx/sites-enabled/discourse.conf
$ sudo rm /etc/nginx/sites-enabled/default
$ sudo service nginx start
```

<<<<<<< HEAD
### Create Production Database

```bash
$ export RAILS_ENV=production
$ rake db:create db:migrate db:seed_fu
```

=======
>>>>>>> ae46ad7005db30cb3a90e85bc1a04a25cd484484
### Deploy Discourse app to /var/www

```bash
$ vi config/initializers/secret_token.rb
$ export RAILS_ENV=production
$ rake assets:precompile
$ sudo -u www-data cp -r ~/discourse/ /var/www
$ sudo -u www-data mkdir /var/www/discourse/tmp/sockets
```

### Start Thin as a daemon listening on domain sockets

```bash
$ cd /var/www/discourse
$ sudo -u www-data thin start -e production -s4 --socket /var/www/discourse/tmp/sockets/thin.sock
```

### Start Sidekiq

```bash
$ sudo -u www-data sidekiq -e production -d -l /var/www/discourse/log/sidekiq.log
```

### Create Discourse admin account

* Logon to site and create account using the application UI
* Now make that account the admin:

```bash
sudo -u www-data rails c     
u = User.first    
u.admin = true    
u.save  
```
Todo: add script to create the admin account

<<<<<<< HEAD
### Edit site settings
=======
### Edit Site Settings
>>>>>>> ae46ad7005db30cb3a90e85bc1a04a25cd484484

The default values are in: app/models/site_setting.rb
* Logon to site with the admin account
* Go to the site settings page: http://discoursetest.org/admin/site_settings
* Set the notification_email. It is the from address used in emails from the system. I set it to info@discoursetest.org.
* Set force_hostname to your domain name. I set it to discoursetest.org. This is used when generating URLs in emails.

## TODO

* Add more information about email configuration and start sidekiq when testing development installation. Should the admin account be set when testing the development server?
* Correct host name in url in emails
* Setup social network login (Is it possible to disable this feature?)
* Add Sam Saffron's Ruby GC tunings
* Add thin and sidekiq as init scripts. I find this cleaner than using bluepill
* Create chef script based on the installation procedure
* Lots of info on server configuration here: http://news.ycombinator.com/item?id=5316093
