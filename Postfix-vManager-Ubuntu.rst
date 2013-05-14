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

  0. What is it?
  1. Features
  2. Screenshots
  3. Requirements
  4. MySQL Server Installation
  5. Postfix and Dovecot
  6. Web Server Installation
  7. Postfix vManager Installation

0. What is it?
==============

The Postfix vManager provides an web based virtual mailbox administration system to allow Unix/Linux administrators to easily manage domains, parking domains, alias domain, mailboxes and forwardres. Postfix vManager is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License version 3.

Postfix vManager was written in PHP, It requires PHP 4 and above, Postfix, Dovecot and MySQL 

1. Features
===========

Standard and enhanced features from Postfix vManager includes:

* Super admin user level with full access.
* Domain Admin user level with access only to assigned domains and their mailboxes and aliases.
* Domain admins can create and modify SubDomain admins and mailboxes.
* JQuery Datatable throughout for quick in browser searching and pagination.
* Create, modify and delete domains including the mailboxes and aliases, a non-super admin can create per-domain; Activate / deactivate mailboxes and aliases at the click of a button.
* Facility for users (mailbox owners) to change their password.
* Parking Domain support.
* Alias domain with virtual alias mailboes support.
* Autoresponder (Vacation) support enabled.

2. Screenshots
==============

Here is some screenshots of Postfix vManager.

3. Requirements
===============

* Postfix which supports virtual domains and mailboxes including vda patch.
* a compatible IMAP/POP3 server (such as Dovecot and Courier), but in this howto we will only discuss Dovecot
* PHP v4 and above with PDO and mysqli supported.

4. MySQL Server Installation
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

5. Postfix and Dovecot
======================

Let's start the Webserver (Apache) installation with PHP support.

::

  sudo apt-get install postfix postfix-mysql sasl2-bin libsasl2-2 libsasl2-modules openssl mailutils procmail dovecot-mysql dovecot-imapd dovecot-pop3d


This will install the Postfix mail server, the Dovecot IMAP and POP daemons, and several supporting packages that provide services related to authentication. You will be prompted to choose a system mail name for Postfix, make sure this should be a fully qualified domain name (FQDN) that points to your server's IP address. I will use example.yourdomain.com in this setup.

After sucessfully installation of Postfix, in the next step will create database for our Postfix vManager.

5.1. Set up MySQL database for Virtual Domains and Users
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



6. WebServer Installation
=========================

Let's start the Webserver (Apache) installation with PHP support.
