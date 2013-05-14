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

Create a forwarding configuration file for forwarding emails from one email address to another, with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

File: /etc/postfix/mysql_virtual_forwarders_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT goto FROM forwarders WHERE address='%s' AND active = '1'


6. WebServer Installation
=========================

Let's start the Webserver (Apache) installation with PHP support.
