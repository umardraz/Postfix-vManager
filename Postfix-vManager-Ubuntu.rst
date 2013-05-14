==========================================================
  Postfix vManager Ubuntu 12.04 LTS Server
==========================================================

:Version: 2.0
:Source: https://github.com/umardraz/Postfix-vManager-Ubuntu
:Keywords: Postfix Mail server management, Cyrus-SASL, Dovecot, MySQL, Virtual Domains, Alias Domains

Author
==========

Copyright (C) Umar Draz <umar_draz@yahoo.com>

Table of Contents
=================

::

  1. Requirements
  2. MySQL Server Installation
  3. Postfix and Dovecot
  4. Web Server Installation
  5. Postfix vManager Installation

1. Requirements
===============

* Postfix which supports virtual domains and mailboxes including vda patch.
* a compatible IMAP/POP3 server (such as Dovecot and Courier), but in this howto we will only discuss Dovecot
* PHP v4 and above with PDO and mysqli supported.

2. MySQL Server Installation
============================

Installing MySQL 5 Server on Ubuntu is a quick and easy process. In classic fashion let’s get the process underway by updating our system.

::

  sudo apt-get update
  sudo apt-get upgrade

Accept any updates that are available to you and then install MySQL Server like so:
  
::

  sudo apt-get install mysql-server mysql-client

The process will not take long but during the installation process you will be prompted to set a password for the MySQL ‘root user’. So choose a strong password and keep it in a safe place for future reference.

When complete, run the following command to secure your installation:

::

  sudo mysql_secure_installation

This utility allows you to limit access to the ‘root’ account, it removes the test database, and allows you to remove all anonymous accounts.

That's All MySQL server has been installed, now lest install Postfix and Dovecot.

3. Postfix and Dovecot
======================

Let's start the Mail server Installation.

::

  sudo apt-get install postfix postfix-mysql sasl2-bin libsasl2-2 libsasl2-modules openssl mailutils procmail dovecot-mysql dovecot-imapd dovecot-pop3d


This will install the Postfix mail server, the Dovecot IMAP and POP daemons, and several supporting packages that provide services related to authentication. You will be prompted to choose a system mail name for Postfix, make sure this should be a fully qualified domain name (FQDN) that points to your server's IP address. I will use example.yourdomain.com in this setup.

After sucessfully installation of Postfix, in the next step will create database for our Postfix vManager.

3.1. Set up MySQL database for Virtual Domains and Users
-----------------

Start the MySQL shell by issuing the following command. You'll be prompted to enter the root password for MySQL that you assigned during the initial setup.

::

  mysql -u root -p

You'll be presented with an interface similar to the following:

::

  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 48
  Server version: 5.5.31-0ubuntu0.12.04.1 (Ubuntu)

  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

  mysql>

Issue the following command to create a database for your mail server and switch to it in the shell:

::

  CREATE DATABASE vmanager;
  USE vmanager;

Create a mail administration user called vadmin and grant it permissions on the mail database with the following commands. Please be sure to replace "vadmin_password" with a password you select for this user.

::

  GRANT SELECT, INSERT, UPDATE, DELETE ON vmanager.* TO 'vadmin'@'localhost' IDENTIFIED BY 'vadmin_password';
  FLUSH PRIVILEGES;

That's all we have sucessfully create database for our application, latter on we will restore our database schema into vmanager database when we will install Postfix vManager.

3.2. Configure Postfix to work with MySQL
-----------------

Create a virtual forwarding file called /etc/postfix/mysql_virtual_forwarders_maps.cf for forwarding emails from one email address to another, with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_virtual_forwarders_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT goto FROM forwarders WHERE address='%s' AND active = '1'

Create a virtual domain configuration file for Postfix called /etc/postfix/mysql_virtual_domains_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_virtual_domains_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT domain FROM domain WHERE domain='%s' and active='1'

Create a virtual mailbox configuration file for Postfix called /etc/postfix/mysql_virtual_mailbox_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_virtual_mailbox_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT CONCAT(domain,'/',maildir) FROM mailbox WHERE username='%s' AND active = '1'

Create a mailbox quota limit configuration file for Postfix called /etc/postfix/mysql_virtual_mailbox_limit_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_virtual_mailbox_limit_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT quota FROM mailbox WHERE username='%s'

Create a sender check configuration file called /etc/postfix/mysql_sender_check.cf so after smtp authentication senders can not use our mail server as open relay.

**File:** /etc/postfix/mysql_sender_check.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT username FROM mailbox WHERE username='%s' and active=1

Create a transport map configuration file called /etc/postfix/mysql_transport.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_transport.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT destination FROM transport where domain = '%s'

Create a alias domains configuration file called /etc/postfix/mysql_virtual_alias_domains_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_virtual_alias_domains_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT target_domain FROM alias_domain WHERE address = '%s' OR address = concat('@', SUBSTRING_INDEX('%s', '@', -1)) AND concat('@', alias_domain) = '%s' AND active = '1'

Create a parking domain configuration file called /etc/postfix/mysql_parking_domains_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_parking_domains_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT domain FROM parking_domains WHERE domain='%s' and active = '1'

Create a virtual groups configuration file called /etc/postfix/mysql_virtual_groups_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_virtual_groups_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT goto FROM groups WHERE address='%s' AND active = '1'

Create a alias domains relay configuration file called /etc/postfix/mysql_alias_domains.maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_alias_domains.maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT DISTINCT alias_domain FROM alias_domain WHERE alias_domain='%s' and active = '1'
  
Set proper permissions and ownership for these configuration files by issuing the following commands:

::

  chmod o= /etc/postfix/mysql_*
  chgrp postfix /etc/postfix/mysql_*

Next, we'll create a user and group for mail handling. All virtual mailboxes will be stored under this user's home directory.

::

  groupadd -g 150 vmail
  useradd -g vmail -u 150 -d /home/vmail -m vmail

Now create /etc/postfix/main.cf with the following contents Please be sure to replace "example.yourdomain.com" with the fully qualified domain name you used for your system mail name.

**File:** /etc/postfix/main.cf

::

  soft_bounce = no
  smtpd_banner = $myhostname
  biff = no
  append_dot_mydomain = no
  inet_interfaces = all
  myhostname = vmanager.cartrade.pk
  myorigin = $myhostname
  mydomain = cartrade.pk
  mynetworks = 127.0.0.0/8
  mynetworks_style = host
  mydestination = $myhostname, localhost.$mydomain, localhost
  alias_maps = $virtual_alias_maps
  local_transport = local
  transport_maps = proxy:mysql:$config_directory/mysql_transport.cf
  debug_peer_level = 2
  debugger_command =
         PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin
         ddd $daemon_directory/$process_name $process_id & sleep 5
  html_directory = /usr/share/doc/postfix
  disable_vrfy_command = yes
  mailbox_size_limit = 0
  owner_request_special = no
  recipient_delimiter = +
  home_mailbox = Maildir/
  mail_owner = postfix
  command_directory = /usr/sbin
  daemon_directory = /usr/lib/postfix
  data_directory = /var/lib/postfix
  queue_directory = /var/spool/postfix
  sendmail_path = /usr/sbin/sendmail
  newaliases_path = /usr/bin/newaliases
  mailq_path = /usr/bin/mailq
  mail_spool_directory = /var/spool/mail
  manpage_directory = /usr/local/man
  setgid_group = postdrop
  unknown_local_recipient_reject_code = 450

  **Virtual Domain**
  virtual_transport = virtual
  virtual_alias_maps =
    proxy:mysql:$config_directory/mysql_virtual_forwarders_maps.cf,
    proxy:mysql:$config_directory/mysql_virtual_groups_maps.cf,
    proxy:mysql:$config_directory/mysql_virtual_alias_domains_maps.cf
  virtual_mailbox_domains = proxy:mysql:$config_directory/mysql_virtual_domains_maps.cf
  virtual_mailbox_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_maps.cf
  virtual_mailbox_limit_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_limit_maps.cf
  virtual_mailbox_base = /home/vmail
  relay_domains =
    proxy:mysql:$config_directory/mysql_parking_domains_maps.cf,
    proxy:mysql:$config_directory/mysql_alias_domains.maps.cf
  proxy_read_maps = $local_recipient_maps $mydestination $virtual_alias_maps $virtual_mailbox_maps $virtual_mailbox_domains $relay_domains $virtual_mailbox_limit_maps $transport_maps
  virtual_minimum_uid = 150
  virtual_uid_maps = static:150
  virtual_gid_maps = static:150

  **Additional for quota support**
  virtual_mailbox_limit_override = yes
  virtual_maildir_limit_message = Sorry, the user's mail quota has exceeded.
  virtual_overquota_bounce = yes


3.3. Configure Dovecot
-----------------

6. WebServer Installation
=========================

Let's start the Webserver (Apache) installation with PHP support.
