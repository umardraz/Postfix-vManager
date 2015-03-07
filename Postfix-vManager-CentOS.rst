==========================================================
  Postfix vManager CentOS 6.4
==========================================================

:Version: 2.0
:Source: https://app.box.com/s/lsz0b4w8cmu46qaz02w39far0djk1uny
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
  6. DKIM Domain Keys

1. Requirements
===============

* Postfix which supports virtual domains and mailboxes including vda patch.
* a compatible IMAP/POP3 server (such as Dovecot and Courier), but in this howto we will only discuss Dovecot
* PHP v4 and above with PDO and mysqli supported.

2. MySQL Server Installation
============================

Installing MySQL 5 Server on CentOS is a quick and easy process. In classic fashion let’s get the process underway by updating our system.

::

  yum update

Accept any updates that are available to you and then install MySQL Server like so:
  
::

  yum install mysql-server mysql-client

If you want to run MySQL by default when the system boots, which is a typical setup, execute the following command:

::

  /sbin/chkconfig --levels 235 mysqld on
  

Now you can start the mysql daemon (mysqld) with the following command (as root):

::

  /etc/init.d/mysqld start

At this point MySQL should be ready to configure and run. While you shouldn't need to change the configuration file, note that it is located at /etc/my.cnf for future reference.

**Configure MySQL and Set Up MySQL databases:**

After installing MySQL, it's recommended that you run mysql_secure_installation, a program that helps secure MySQL. While running mysql_secure_installation, you will be presented with the opportunity to change the MySQL root password, remove anonymous user accounts, disable root logins outside of localhost, and remove test databases. It is recommended that you answer yes to these options. If you are prompted to reload the privilege tables, select yes. Run the following command to execute the program:

::

  mysql_secure_installation

That's All MySQL server has been installed, now lest install Postfix and Dovecot.

3. Postfix and Dovecot
======================

Let's start the Mail server Installation.

::

  yum remove sendmail
  yum install cyrus-sasl cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-md5 cyrus-sasl-plain dovecot dovecot-mysql


This will install the Postfix mail server, the Dovecot IMAP and POP daemons, and several supporting packages that provide services related to authentication.

3.1. Apply The Quota Patch To Postfix
-------------------------------------

First we need to install some prerequisite packages.

::

  yum install rpm-build db4-devel zlib-devel openldap-devel pcre-devel mysql-devel openssl-devel gcc

We have to get the Postfix source rpm, patch it with the quota patch, build a new Postfix rpm package and install it. 

::

  cd /usr/src
  wget http://vault.centos.org/6.4/os/Source/SPackages/postfix-2.6.6-2.2.el6_1.src.rpm
  rpm -ivh postfix-2.6.6-2.2.el6_1.src.rpm

The last command will show some warnings that you can ignore:

::

  cd /root/rpmbuild/SOURCES
  wget http://vda.sourceforge.net/VDA/postfix-2.6.5-vda-ng.patch.gz
  gunzip postfix-2.6.5-vda-ng.patch.gz
  cd /root/rpmbuild/SPECS/

Now we must edit the file postfix.spec:
  
::

  vi postfix.spec


Add Patch0: postfix-2.6.5-vda-ng.patch to the # Patches stanza, and %patch0 -p1 -b .vda-ng to the %setup -q stanza:

::

  [...]
  # Patches

  Patch0: postfix-2.6.5-vda-ng.patch
  Patch1: postfix-2.6.1-config.patch
  Patch2: postfix-2.6.1-files.patch
  Patch3: postfix-alternatives.patch
  Patch8: postfix-large-fs.patch
  Patch9: pflogsumm-1.1.1-datecalc.patch
  Patch10: postfix-2.6.6-CVE-2011-0411.patch
  Patch11: postfix-2.6.6-CVE-2011-1720.patch
  [...]
  %prep
  %setup -q
  # Apply obligatory patches
  %patch0 -p1 -b .vda-ng
  %patch1 -p1 -b .config
  %patch2 -p1 -b .files
  %patch3 -p1 -b .alternatives
  %patch8 -p1 -b .large-fs
  [...]

Then we build our new Postfix rpm package with quota and MySQL support:

::

  rpmbuild -ba postfix.spec

Our Postfix rpm package is created in /root/rpmbuild/RPMS/x86_64 (/root/rpmbuild/RPMS/i386 if you are on an i386 system), so we go there:

::

  cd /root/rpmbuild/RPMS/x86_64
  ls -al

shows you the available packages:

::

  drwxr-xr-x 2 root root     4096 May 18 21:57 .
  drwxr-xr-x 3 root root     4096 May 18 21:56 ..
  -rw-r--r-- 1 root root 11573873 May 18 21:57 postfix-2.6.6-2.2.el6.x86_64.rpm
  -rw-r--r-- 1 root root    63421 May 18 21:57 postfix-perl-scripts-2.6.6-2.2.el6.x86_64.rpm

To make sure that no version of postfix was previously installed on your system, use:

::

  yum remove postfix
  
Pick the Postfix package and install it like this:

::

  rpm -ivh postfix-2.6.6-2.2.el6.x86_64.rpm

The above command will install new postfix package with quota enabled pacakge.

2. Set up MySQL database for Virtual Domains and Users
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

Create a mail administration user called vadmin and grant it permissions on the mail database with the following commands. Please be sure to replace "vadmin_password" with a password you select for this user.

::

  GRANT SELECT, INSERT, UPDATE, DELETE ON vmanager.* TO 'vadmin'@'localhost' IDENTIFIED BY 'vadmin_password';
  FLUSH PRIVILEGES;

That's all we have sucessfully create database for our application, latter on we will restore our database schema into vmanager database when we will install Postfix vManager.

3.3. Configure Postfix to work with MySQL
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
  query = SELECT username FROM ( SELECT username as username FROM mailbox UNION ALL SELECT address FROM alias_domain) a where username = '%s'

Create a transport map configuration file called /etc/postfix/mysql_transport.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /etc/postfix/mysql_transport.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT destination FROM transport where domain = '%s'

Create an alias domains configuration file called /etc/postfix/mysql_virtual_alias_domains_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

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

Create an alias domains relay configuration file called /etc/postfix/mysql_alias_domains.maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

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
  myhostname = example.yourdomain.com
  myorigin = $myhostname
  mydomain = yourdomain.com
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
  html_directory = no
  disable_vrfy_command = yes
  mailbox_size_limit = 0
  owner_request_special = no
  recipient_delimiter = +
  home_mailbox = Maildir/
  mail_owner = postfix
  command_directory = /usr/sbin
  daemon_directory = /usr/libexec/postfix
  data_directory = /var/lib/postfix
  queue_directory = /var/spool/postfix
  sendmail_path = /usr/sbin/sendmail
  newaliases_path = /usr/bin/newaliases
  mailq_path = /usr/bin/mailq.postfix
  mail_spool_directory = /var/spool/mail
  manpage_directory = /usr/share/man
  setgid_group = postdrop
  unknown_local_recipient_reject_code = 450

  # Virtual Domains and Users
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

  # Additional for quota support
  virtual_mailbox_limit_override = yes
  virtual_maildir_limit_message = Sorry, the user's mail quota has exceeded.
  virtual_overquota_bounce = yes

  # SMTP Authentication 
  smtpd_sasl_auth_enable = yes
  smtpd_sasl_security_options = noanonymous
  broken_sasl_auth_clients = yes
  smtpd_sasl_authenticated_header = yes
  smtpd_sasl_type = dovecot
  smtpd_sasl_path = private/auth

  # TLS/SSL
  smtpd_use_tls = yes
  smtpd_tls_auth_only = no
  smtpd_tls_cert_file = /etc/postfix/smtpd.cert
  smtpd_tls_key_file = /etc/postfix/smtpd.key

  # Other Configurations
  strict_rfc821_envelopes = yes
  smtpd_soft_error_limit = 10
  smtpd_hard_error_limit = 20
  smtpd_data_restrictions = reject_unauth_pipelining, reject_multi_recipient_bounce
  smtpd_etrn_restrictions = reject
  smtpd_helo_required = yes
  smtpd_recipient_limit = 25
  #smtpd_sender_login_maps = mysql:$config_directory/mysql_sender_check.cf

  smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_invalid_hostname,
    reject_unauth_pipelining,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    reject_non_fqdn_recipient,
    reject_unknown_recipient_domain,
    permit
  
  smtpd_sender_restrictions =
    permit_mynetworks,
    reject_unverified_sender,
    #reject_sender_login_mismatch,
    #reject_unauthenticated_sender_login_mismatch,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    permit

This completes the configuration for Postfix. Next, you'll make an SSL certificate for the Postfix server that contains values appropriate for your organization.

Create an SSL Certificate for Postfix
-----------------

Issue the following commands to create the SSL certificate

::

  cd /etc/postfix
  openssl req -new -outform PEM -out smtpd.cert -newkey rsa:2048 -nodes -keyout smtpd.key -keyform PEM -days 365 -x509

You will be asked to enter several values similar to the output shown below. Be sure to enter the fully qualified domain name you used for the system mailname in place of "example.yourdomain.com".

::

  Country Name (2 letter code) [AU]:PK
  State or Province Name (full name) [Some-State]:Punjab
  Locality Name (eg, city) []:Lahore
  Organization Name (eg, company) [Internet Widgits Pty Ltd]:MyComapny
  Organizational Unit Name (eg, section) []:Email Services
  Common Name (eg, YOUR name) []:example.yourdomain.com
  Email Address []:webmaster@yourdomain.com

Set proper permissions for the key file by issuing the following command:

::

  chmod o= /etc/postfix/smtpd.key

This completes SSL certificate creation for Postfix. Next, you'll need to configure Dovecot for imap service.

3.4. Configure Dovecot
-----------------

Replace the contents of the file with the following example, substituting your system's domain name for yourdomain.com.

**File:** /etc/dovecot/dovecot.conf

::

  auth_mechanisms = plain login
  base_dir = /var/run/dovecot/
  disable_plaintext_auth = no
  first_valid_gid = 150
  first_valid_uid = 150
  last_valid_gid = 150
  last_valid_uid = 150
  log_path = /var/log/mail.log
  log_timestamp = %Y-%m-%d %H:%M:%S
  auth_username_format = %Lu
  mail_access_groups = mail
  mail_location = maildir:~/Maildir

  passdb {
    args = /etc/dovecot/dovecot-mysql.conf
    driver = sql
  }

  protocols = imap

  service auth {
    unix_listener /var/spool/postfix/private/auth {
      group = postfix
      mode = 0660
      user = postfix
    }
  }

  service imap-login {
    inet_listener imap {
      address = *
      port = 143
    }
  }

  service pop3-login {
    inet_listener pop3 {
      address = *
      port = 110
    }
  }

  ssl = yes
  ssl_cert = </etc/postfix/smtpd.cert
  ssl_key = </etc/postfix/smtpd.key

  userdb {
    args = /etc/dovecot/dovecot-mysql.conf
    driver = sql
  }

MySQL will be used to store password information, so /etc/dovecot/dovecot-mysql.conf must be edited. Replace the contents of the file with the following example, making sure to replace "vadmin_password" with your mail password.

**File:** /etc/dovecot/dovecot-mysql.conf

::

  driver = mysql
  connect = host=localhost user=vadmin password=vadmin_password dbname=vmanager
  default_pass_scheme = MD5-CRYPT
  password_query = SELECT password FROM mailbox WHERE username = '%u'
  user_query = SELECT '/home/vmail/%d/%n/Maildir' as home, 'maildir:/home/vmail/%d/%n/Maildir' as mail, 150 AS uid, 150 AS gid, concat('dirsize:storage=',quota) AS quota FROM mailbox WHERE username ='%u' AND active ='1'

Dovecot has now been configured. You must restart it to make sure it is working properly, also restart postfix:

::

  service dovecot restart
  service postfix restart
  
That's Postfix and Dovecot installation is completed. Now let's install Apache and PHP for Postfix vManager Application.

Testing TLS
===========

To verify Postfix supports TLS, it has to be displaying STARTTLS when you connect to port 25 with telnet and run the EHLO command. We set this up in a previous step.

To verify the SSL certificate is working and Postfix can negotiate the SSL encryption you can use the openssl command.

::

  openssl s_client -starttls smtp -crlf -connect mail.example.com:25

Substitute mail.example.com with the hostname of your mail server.

4. WebServer Installation
=========================

Apache is easily installed by entering the following command.

::

  yum install httpd

**Configure Name-based Virtual Hosts**

There are different ways to set up Virtual Hosts, however we recommend the method below. By default, Apache listens on all IP addresses available to it.

Now we will create virtual host entries for example.yourdomain.com site that we need to host with this server. Here is this.

**File:** /etc/httpd/conf.d/vhost.conf

::

  NameVirtualHost *:80
  <VirtualHost *:80>
     ServerAdmin webmaster@yourdomain.com
     ServerName yourdomain.com
     ServerAlias example.yourdomain.com
     DocumentRoot /var/www/vmanager
     ErrorLog /var/log/httpd/error.log
     CustomLog /var/log/httpd/access.log combined
  </VirtualHost>

Before you can use the above configuration you'll need to create the specified directories. For the above configuration, you can do this with the following commands:

::

  mkdir -p /var/www/vmanager

Postfix vManager depends on url rewriting for SEO purpose. In order to take advantage of this feature we need to edit httpd.conf file as follows.

Edit /etc/httpd/conf/httpd.conf file and change **AllowOverride None** to **AllowOverride All** under / directory e.g.

::

  <Directory />
    Options FollowSymLinks
    AllowOverride All
  </Directory>

After you've set up your virtual hosts, issue the following command to run Apache for the first time:

::

  /etc/init.d/httpd restart
  
If you want to run Apache by default when the system boots, which is a typical setup, execute the following command:

::

  /sbin/chkconfig --levels 235 httpd on
  
Installing PHP
-----------------

We will therefore install PHP with the following command.

::

  yum install php php-mysql php-pdo php-mysqli php-mbstring php-pear

Once PHP5 is installed we'll need to tune the configuration file located in /etc/php.ini to enable more descriptive errors, logging, and better performance. These modifications provide a good starting point if you're unfamiliar with PHP configuration.

Make sure that the following values are set, and relevant lines are uncommented (comments are lines beginning with a semi-colon (;)):

**File:** /etc/php.ini

::

  error_reporting = E_COMPILE_ERROR|E_RECOVERABLE_ERROR|E_ERROR|E_CORE_ERROR
  display_errors = Off
  log_errors = On
  error_log = /var/log/php.log
  max_execution_time = 300
  memory_limit = 64M
  register_globals = Off

Whenver you change anything in php.ini file then you need to rstart apache server.

::

  /etc/init.d/httpd restart

5. Postfix vManager
===================

First download postfix vmanager source from this url :Source: https://app.box.com/s/lsz0b4w8cmu46qaz02w39far0djk1uny

After downloading the postfix-vmanager-2.0.tar.gz just extract the source. 

Then first remove the /var/www/vmanager directory and move extracted source into /var/www/vmanager/ let's do it.

::

  tar xzvpf postfix-vmanager-2.0.tar.gz
  rm -rf /var/www/vmanager
  mv postfix-vmanager-2.0 /var/www/vmanager
  
Next restore the database, with the following command

::

  cd /var/www/vmanager/  
  mysql -uroot -proot_pass vmanager < setup/vmanager.sql

5.1. Configure Postfix vManager
----------------------

Edit the inc/config.inc.php file and add your settings there. The most important settings are those for your database server.

::

  $CONF['database_host'] = 'localhost';
  $CONF['database_user'] = 'vadmin';
  $CONF['database_password'] = 'vadmin_password';
  $CONF['database_name'] = 'vmanager';
  $CONF['database_port'] = '3306';
  $CONF['database_prefix'] = '';

Postfix vManager require write access to its directory. So you need to change the vmanager directory ownership with that user as web server running.

::

  chown -R apache:apache /var/www/vmanager/

5.2. Check settings, and create Admin user
------------------------------------------

Hit :Source: https://example.yourdomain.com/ in a web browser. You should see a list of 'OK' messages. Otherwise reslove the issue if found. 

Create the admin user using the form displayed. This is all that is needed.

5.3. Vacations
--------------

The vacation script runs as service within Postfix's master.cf configuration file. Mail is sent to the vacation service via a transport table mapping. When users mark themselves as away on vacation, an alias is added to their account sending a copy of all mail to them to the vacation service.

To use vacation services you need to first create vacation domain. Just login as Super Admin account and then 

5.4. Installing Vacations
-------------------------

Login as Super Admin and then create Vacation domain following this.

::

  Go to Settings -> Vacation Domain.

There are a bunch of Perl modules which we need to install for Vacation setup. We need to first install epel rpm package to install these perl modules.

Let's install epel rpm if its not already installed.

::

    wget http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
    rpm -Uvh epel-release-6-8.noarch.rpm

After installing epel rpm in the next step we will install perl modules.

::

  yum install perl-MIME-EncWords perl-Email-Valid perl-Mail-Sender perl-Log-Log4perl perl-MIME-Charset

**Create Vacation Account:**

Create a dedicated local user account called "vacation". This user handles all potentially dangerous mail content - that is why it should be a separate account.

Do not use "nobody", and most certainly do not use "root" or "postfix". The user will never log in, and can be given a "*" password and non-existent shell and home directory.

Create the user with the following command.

::

  useradd vacation -c "Vacation Owner" -d /nonnonexistent -s /bin/false

**Create a directory:**

Create a directory, for example  /var/spool/vacation, that is accessible only to the "vacation" user. This is where the vacation script is supposed to store its temporary files. 

::

  mkdir /var/spool/vacation
  
**Copy Files:**

Copy the vacation.pl file to the directory you created above:

::

  cp setup/vacation.pl /var/spool/vacation/vacation.pl
  chown -R vacation:vacation /var/spool/vacation/
  
Which will then look something like:

::

  -rwx------   1 vacation  vacation  3356 Dec 21 00:00 vacation.pl*

**Setup the transport type:**

Define the transport type in the Postfix /etc/postfix/master.cf file:

::

  vacation    unix  -       n       n       -       -       pipe
    flags=Rq user=vacation argv=/var/spool/vacation/vacation.pl -f ${sender} -- ${recipient}
    
Here we need to restart postfix service.

::

  service postfix restart

**Configure vacation.pl"**

The perl /var/spool/vacation/vacation.pl script needs to know which database you are using, and also how to connect to the database.

Change any variables starting with '$db_' and '$db_type'

Change the $vacation_domain variable to match what you entered through your Super Admin login.

Here is the example of vacatino.pl settings for database and domain name

::

  our $db_type = 'mysql';
  our $db_host = 'localhost';
  our $db_username = 'username';
  our $db_password = 'password';
  our $db_name     = 'dbname';
  our $vacation_domain = 'autoreply.yourdomain.com';

Done! When this is all in place you need to have a look at the Postfix vManager inc/config.inc.php. Here you need to enable Virtual Vacation for the site.

6. DKIM Domain Keys
===================

DomainKeys Identified Mail (DKIM) is a method for associating a domain name to an email message, thereby allowing a person, role, or organization to claim some responsibility for the message and helps verify that your mail is legitimate. This will help your email not get flagged a spam or fraud, especially if you are doing bulk emailing or important emails.

First, install EPEL and dkim-milter

::

  rpm -Uvh http://epel.mirror.net.in/epel/6/x86_64/epel-release-6-8.noarch.rpm
  yum install dkim-milter
  
Setup a domain key for your domain e.g yourdomain.com

::

  DKIMDOMAIN=yourdomain.com
  mkdir -p /etc/dkim/keys/$DKIMDOMAIN
  cd /etc/dkim/keys/$DKIMDOMAIN
  dkim-genkey -r -d $DKIMDOMAIN
  mv default.private default

If you want an easy web based way check out http://www.socketlabs.com/services/dkwiz which also gives you the DNS records.

Create a file **/etc/dkim-keys.conf** and insert into it a line like this (replacing 'domain.com' with your own domain)

::
  
  *@yourdomain.com:yourdomain.com:/etc/dkim/keys/yourdomain.com/default

If you used command line then check the file at /etc/dkim/keys/yourdomain.com/default.txt which will have something like this

::

  default._domainkey IN TXT "v=DKIM1; k=rsa; p=MIGfMA0frgfrefgrweferNYlS+8jyrbAxNsghsPrWYgOQQWI0Ab4e9MT" ; ----- DKIM default for yourdomain.com

Yours should be much longer, this was snipped for brevity. You need to add the TXT record **default._domainkey** with the key between the quotes. If you are using standard bind then you can copy/paste that into the named file.

Another TXT record worth adding is

::

  _domainkey IN TXT t=y;o=~;
  
Now look for and edit your **/etc/mail/dkim-milter/dkim-filter.conf**

You need to have 2 lines like this.

::

  KeyList /etc/dkim-keys.conf
  Socket inet:8891@localhost

Then restart the DKIM filter

::

  /etc/init.d/dkim-filter restart
  
Now add the following code into the postifx config. This goes into main.cf (/etc/postfix/main.cf )

::

  milter_default_action = accept
  milter_protocol = 2
  smtpd_milters = inet:localhost:8891
  non_smtpd_milters = inet:localhost:8891

Then of course restart postfix

::

  postfix reload
  
This should now sign emails going out with the domain key.

It pays to use this webpage to check things are working http://www.brandonchecketts.com/emailtest.php

You can also check your domain TXT record verification from here: http://dkimcore.org/tools/keycheck.html
