Documentation of setting up GitLab6 on SLES11.2
===============================================

derived from official installation guide, for more information:
https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/installation.md


Install needed Packages and Dependencies
----------------------------------------

You need the SDK Repository to make a clean install. Be sure you have added this one.

run as root

```
zypper clean && zypper refresh
zypper install --type pattern Basis-Devel
zypper install sudo vim vim-data ctags postfix python ntp git

zypper in mysql mysql-tools libxml2-devel libmysqlclient-devel
zypper rm nginx-1.0 ruby
zypper in curl git-core findutils-locate
zypper in libopenssl-devel zlib-devel libgpg-error-devel
```

Install libicu and libicu-devel, version > 4.2 is necessary!

```
wget ftp://rpmfind.net/linux/opensuse/update/11.3/rpm/x86_64/libicu-4.2-7.3.1.x86_64.rpm
rpm -i libicu-4.2-7.3.1.x86_64.rpm
wget ftp://rpmfind.net/linux/opensuse/update/11.3/rpm/x86_64/libicu-devel-4.2-7.3.1.x86_64.rpm
rpm -i libicu-devel-4.2-7.3.1.x86_64.rpm
```

Build Ruby 2, this version is not in the official repos

```
mkdir tmp_inst; cd !$
wget ftp://ftp.ruby-lang.org/pub/ruby/2.0/ruby-2.0.0-p247.tar.gz
tar xfvz ruby-2.0.0-p247.tar.gz
cd ruby-2.0.0-p247

		This will take some time

./configure && make && make install

link ruby to /usr/bin/ruby
```

build libxslt from hand:


```
wget ftp://xmlsoft.org/libxml2/libxslt-1.1.28.tar.gz
tar xfvz libxslt-1.1.28.tar.gz
cd libxslt-1.1.28/
./configure --prefix=/usr
make && make install

```

build openssl extension for ruby

```	
cd ext/openssl
ruby extconf.rb
make && make install
ln -s /usr/local/bin/gem /usr/bin/gem
gem update --system
gem install bundler

```

Build needed gems:

```
gem install charlock_holmes --version '0.6.9.4'
gem install nokogiri -v '1.5.10'
```

Install and configure redis

```
wget http://download.redis.io/releases/redis-2.6.16.tar.gz
tar xfvz redis-2.6.16.tar.gz
cd redis-2.6.16
make

edit /etc/sysctl.conf and insert
		vm.overcommit_memory = 1

mv redis-2.6.16 /opt/redis
link redis binaries to to /usr/bin
ln -s /opt/redis/src/redis-cli /usr/bin/redis-cli
ln -s /opt/redis/src/redis-server /usr/bin/redis-server
```

Create a group and a user 'git'

```
groupadd git
useradd -m -G git --system -s /bin/sh -c 'GitLab' -d /home/git git
```

Install and configure Gitlab-Shell
----------------------------------

```
cd /home/git
su git
git config --global http.sslVerify false
git clone https://github.com/gitlabhq/gitlab-shell.git
cd /home/git/gitlab-shell
git checkout v1.7.1
cp config.yml.example config.yml
edit config and replace github url, if necessary
setup gitlab shell:
./bin/install
exit (change back to admin user)
```

Database Setup (MySQL)
----------------------

This section is taken from official Gitlab Documentation.

Secure your installation.
```
mysql_secure_installation
```

Login to MySQL

```
mysql -u root -p
```

Type the database root password

Create a user for GitLab
do not type the 'mysql>', this is part of the prompt
change $password in the command below to a real password you pick

```mysql
mysql> CREATE USER 'gitlab'@'localhost' IDENTIFIED BY '$password';
```
 
Create the GitLab production database
-------------------------------------

```mysql
mysql> CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
```
Grant the GitLab user necessary permissions on the table.

```mysql
mysql> GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'gitlab'@'localhost';
```
Quit the database session
```mysql
mysql> \q
```

Try connecting to the new database with the new user

```
sudo -u git -H mysql -u gitlab -p -D gitlabhq_production
```

Type the password you replaced $password with earlier

You should now see a 'mysql>' prompt

Quit the database session

```mysql
mysql> \q
```

You are done installing the database and can go back to the rest of the installation.

Checkout Gitlab and create needed directories
---------------------------------------------

*as user git*:

```
git clone https://github.com/gitlabhq/gitlabhq.git gitlab
cd /home/git/gitlab
git checkout 6-1-stable

cp config/gitlab.yml.example config/gitlab.yml
change localhost to fully qualified domain name in gitlab.yaml
```

make sure gitlab can read/write in in/to log and tmp dirs

as root:

```
chown -R git log/
chown -R git tmp/
chmod -R u+rwX  log/
chmod -R u+rwX  tmp/
```

as git:

```
mkdir /home/git/gitlab-satellites
```

Create directories for sockets/pids and make sure GitLab can write to them

```
sudo -u git -H mkdir tmp/pids/
sudo -u git -H mkdir tmp/sockets/
sudo chmod -R u+rwX  tmp/pids/
sudo chmod -R u+rwX  tmp/sockets/
```

Enable cluster mode if you expect to have a high load instance
Ex. change amount of workers to 3 for 2GB RAM server

```
sudo -u git -H editor config/unicorn.rb
(has been set to 5)
```

Configure git
-------------

Configure Git global settings for git user, useful when editing via web
Edit user.email according to what is set in gitlab.yml

```
sudo -u git -H git config --global user.name "GitLab"
sudo -u git -H git config --global user.email "gitlab@localhost"
sudo -u git -H git config --global core.autocrlf input
```

Configure Database
------------------

as git:

```
cp config/database.yml.mysql config/database.yml
```

put login and password for database to
```
gitlab/config/database.yml
```
and change root to gitlab

Make config/database.yml readable to git only

```
sudo -u git -H chmod o-rwx config/database.yml
```


Now install all other needed gems. 

Note it says: without ... postgres

```
bundle install --deployment --without development test postgres aws
```

start redis as daemon via 
```
redis-server &
```
Set up Gitlab
-------------

Execute Setup for Gitlab
```
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
```
type yes, if everything went fine, output shall be:

> [..]
> Administrator account created:
>
> login.........admin@local.host
> password......5iveL!fe

Copy binaries to init.d
```
sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
sudo chmod +x /etc/init.d/gitlab
```

and make Gitlab start on boot
```
chkconfig -add gitlab
```

check application status

as git
```
bundle exec rake gitlab:env:info RAILS_ENV=production
```
Sample Output:
```
System information
System:		SUSE LINUX 11
Current User:	git
Using RVM:	no
Ruby Version:	2.0.0p247
Gem Version:	2.1.5
Bundler Version:1.3.5
Rake Version:	10.1.0

GitLab information
Version:	6.1.0
Revision:	b595503
Directory:	/home/git/gitlab
DB Adapter:	mysql2
URL:		http://localhost
HTTP Clone URL:	http://localhost/some-project.git
SSH Clone URL:	git@localhost:some-project.git
Using LDAP:	no
Using Omniauth:	no

GitLab Shell
Version:	1.7.1
Repositories:	/home/git/repositories/
Hooks:		/home/git/gitlab-shell/hooks/
Git:		/usr/bin/git
```

as root start gitlab
```
/etc/init.d/gitlab restart
```

Sample output:
```
Starting the GitLab Unicorn web server...
Starting the GitLab Sidekiq event dispatcher...
The GitLab Unicorn webserver with pid 30332 is running.
The GitLab Sidekiq job dispatcher with pid 30358 is running.
GitLab and all it's components are up and running.
```
double check environment

as git

```
bundle exec rake gitlab:check RAILS_ENV=production
```

if everything is green -> everything went fine

Sample output:
```
$> time bundle exec rake gitlab:check RAILS_ENV=production
Checking Environment ...

Git configured for git user? ... yes
Has python2? ... yes
python2 is supported version? ... yes

Checking Environment ... Finished

Checking GitLab Shell ...

GitLab Shell version >= 1.7.1 ? ... OK (1.7.1)
Repo base directory exists? ... yes
Repo base directory is a symlink? ... no
Repo base owned by git:git? ... yes
Repo base access is drwxrws---? ... yes
update hook up-to-date? ... yes
update hooks in repos are links: ... can't check, you have no projects

Checking GitLab Shell ... Finished

Checking Sidekiq ...

Running? ... yes

Checking Sidekiq ... Finished

Checking GitLab ...

Database config exists? ... yes
Database is SQLite ... no
All migrations up? ... yes
GitLab config exists? ... yes
GitLab config outdated? ... no
Log directory writable? ... yes
Tmp directory writable? ... yes
Init script exists? ... yes
Init script up-to-date? ... yes
projects have namespace: ... can't check, you have no projects
Projects have satellites? ... can't check, you have no projects
Redis version >= 2.0.0? ... yes
Your git bin path is "/usr/bin/git"
Git version >= 1.7.10 ? ... yes (1.7.12)

Checking GitLab ... Finished


real	0m16.666s
user	0m14.009s
sys	0m2.128s
```

NGINX
-----

install not from repo as this version is too old
```
wget http://download.opensuse.org/repositories/home:/itcrow:/branches:/home:/emendonca/SLE_11_SP2/x86_64/nginx-1.2.3-12.1.x86_64.rpm
rpm -i nginx-1.2.3-12.1.x86_64.rpm

cd /etc/nginx
mkdir sites-available
mkdir sites-enabled
add folger 'sites-enabled' to http-section in nginx.conf
```

```
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;
-->    include /etc/nginx/sites-enabled/*; <---
    #gzip  on;

    server {
        listen
				[..]
```

copy gitlab-webdirectory to nginx and link it to the sites-enabled directory
```
cd /home/git/gitlab
sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
```
replace default nginx user by user git and group root, in /etc/nginx/nginx.conf
```
#user nginx:
user git root;
```

Restart nginx
```
sudo /etc/init.d/nginx restart
```
