Postfix
========

介绍
-----

Postfix是一个邮件传输客户端(MTA),它也是ubuntu中默认的邮件传输客户端.它是Ubuntu的main软件库中的一个软件.这意味着它拥有安全更新.这份指南告诉你如何安装及配置postfix并将其设置成一个使用安全连接的SMTP服务器.

什么是邮件传输代理
-------------------

In other words, it's a mail server not a mail client like Thunderbird, Evolution, Outlook, Eudora, or a web-based email service like Yahoo, GMail, Hotmail, Earthlink, Comcast, SBCGlobal.net, ATT.net etc.... If you worked for a company named Acme and owned acme.com, you could provide your employees with email addresses @acme.com. Employees could send an receive email through your computer, but not without your computer running all the time. If all your email addresses are at a domain (@gmail.com, @yahoo.com) you do not own (you don't own Google) or do not host (acme.com) then you do not need this at all. 

安装
-----

In order to install Postfix with SMTP-AUTH and TLS, first install the postfix package from the Main repository using your favorite package manager. For example:

.. code-block:: sh

   sudo apt-get install postfix

Simply accept the defaults when the installation process asks questions. The configuration will be done in greater detail in the next stage.

配置
-----

From a terminal prompt:

.. code-block:: sh

   sudo dpkg-reconfigure postfix

Insert the following details when asked (replacing server1.example.com with your domain name if you have one):

#. General type of mail configuration: Internet Site
#. NONE doesn't appear to be requested in current config
#. System mail name: server1.example.com
#. Root and postmaster mail recipient: <admin_user_name>
#. Other destinations for mail: server1.example.com, example.com, localhost.example.com, localhost
#. Force synchronous updates on mail queue?: No
#. Local networks: 127.0.0.0/8
#. Yes doesn't appear to be requested in current config
#. Mailbox size limit (bytes): 0
#. Local address extension character: +
#. Internet protocols to use: all

Now is a good time to decide which mailbox format you want to use. By default Postifx will use mbox for the mailbox format. Rather than editing the configuration file directly, you can use the postconf command to configure all postfix parameters. The configuration parameters will be stored in /etc/postfix/main.cf file. Later if you wish to re-configure a particular parameter, you can either run the command or change it manually in the file.

To configure the mailbox format for Maildir:

.. code-block:: sh

   sudo postconf -e 'home_mailbox = Maildir/'

You may need to issue this as well:

.. code-block:: sh

   sudo postconf -e 'mailbox_command ='

.. note::

   This will place new mail in /home/username/Maildir so you will need to configure your Mail Delivery Agent to use the same path.

Configure Postfix to do SMTP AUTH using SASL (saslauthd):

.. code-block:: sh

   sudo postconf -e 'smtpd_sasl_local_domain ='
   sudo postconf -e 'smtpd_sasl_auth_enable = yes'
   sudo postconf -e 'smtpd_sasl_security_options = noanonymous'
   sudo postconf -e 'broken_sasl_auth_clients = yes'
   sudo postconf -e 'smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination'
   sudo postconf -e 'inet_interfaces = all'

下一步编辑 /etc/postfix/sasl/smtpd.conf 添加以下行:

.. code-block:: ini

   pwcheck_method: saslauthd
   mech_list: plain login

Generate certificates to be used for TLS encryption and/or certificate Authentication:

.. code-block:: sh

   touch smtpd.key
   chmod 600 smtpd.key
   openssl genrsa 1024 > smtpd.key
   openssl req -new -key smtpd.key -x509 -days 3650 -out smtpd.crt # has prompts
   openssl req -new -x509 -extensions v3_ca -keyout cakey.pem -out cacert.pem -days 3650 # has prompts
   sudo mv smtpd.key /etc/ssl/private/
   sudo mv smtpd.crt /etc/ssl/certs/
   sudo mv cakey.pem /etc/ssl/private/
   sudo mv cacert.pem /etc/ssl/certs/

Configure Postfix to do TLS encryption for both incoming and outgoing mail:

.. code-block:: sh

   sudo postconf -e 'smtp_tls_security_level = may'
   sudo postconf -e 'smtpd_tls_security_level = may'
   sudo postconf -e 'smtpd_tls_auth_only = no'
   sudo postconf -e 'smtp_tls_note_starttls_offer = yes'
   sudo postconf -e 'smtpd_tls_key_file = /etc/ssl/private/smtpd.key'
   sudo postconf -e 'smtpd_tls_cert_file = /etc/ssl/certs/smtpd.crt'
   sudo postconf -e 'smtpd_tls_CAfile = /etc/ssl/certs/cacert.pem'
   sudo postconf -e 'smtpd_tls_loglevel = 1'
   sudo postconf -e 'smtpd_tls_received_header = yes'
   sudo postconf -e 'smtpd_tls_session_cache_timeout = 3600s'
   sudo postconf -e 'tls_random_source = dev:/dev/urandom'
   sudo postconf -e 'myhostname = server1.example.com' # remember to change this to yours

The file /etc/postfix/main.cf should now look like this:

.. code-block:: ini

   # See /usr/share/postfix/main.cf.dist for a commented, more complete version

   smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
   biff = no

   # appending .domain is the MUA's job.
   append_dot_mydomain = no

   # Uncomment the next line to generate "delayed mail" warnings
   #delay_warning_time = 4h

   myhostname = server1.example.com
   alias_maps = hash:/etc/aliases
   alias_database = hash:/etc/aliases
   myorigin = /etc/mailname
   mydestination = server1.example.com, example.com, localhost.example.com, localhost
   relayhost =
   mynetworks = 127.0.0.0/8
   mailbox_command = procmail -a "$EXTENSION"
   mailbox_size_limit = 0
   recipient_delimiter = +
   inet_interfaces = all
   smtpd_sasl_local_domain =
   smtpd_sasl_auth_enable = yes
   smtpd_sasl_security_options = noanonymous
   broken_sasl_auth_clients = yes
   smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination
   smtpd_tls_auth_only = no
   #Use these on Postfix 2.2.x only
   #smtp_use_tls = yes
   #smtpd_use_tls = yes
   #For Postfix 2.3 or above use:
   smtp_tls_security_level = may
   smtpd_tls_security_level = may
   smtp_tls_note_starttls_offer = yes
   smtpd_tls_key_file = /etc/ssl/private/smtpd.key
   smtpd_tls_cert_file = /etc/ssl/certs/smtpd.crt
   smtpd_tls_CAfile = /etc/ssl/certs/cacert.pem
   smtpd_tls_loglevel = 1
   smtpd_tls_received_header = yes
   smtpd_tls_session_cache_timeout = 3600s
   tls_random_source = dev:/dev/urandom

Restart the postfix daemon like this:

.. code-block:: sh

   sudo /etc/init.d/postfix restart

身份验证
--------

The next steps are to configure Postfix to use SASL for SMTP AUTH.

First you will need to install the libsasl2-2, sasl2-bin and libsasl2-modules from the Main repository [i.e. sudo apt-get install them all].

.. note::

   if you are using Ubuntu 6.06 (Dapper Drake) the package name is libsasl2.

We have to change a few things to make it work properly. Because Postfix runs chrooted in /var/spool/postfix we have change a couple paths to live in the false root. (ie. /var/run/saslauthd becomes /var/spool/postfix/var/run/saslauthd):


.. note::

   by changing the saslauthd path other applications that use saslauthd may be affected. 

First we edit /etc/default/saslauthd in order to activate saslauthd. Remove # in front of START=yes, add the PWDIR, PARAMS, and PIDFILE lines and edit the OPTIONS line at the end:

.. code-block:: ini

   # This needs to be uncommented before saslauthd will be run automatically
   START=yes

   PWDIR="/var/spool/postfix/var/run/saslauthd"
   PARAMS="-m ${PWDIR}"
   PIDFILE="${PWDIR}/saslauthd.pid"

   # You must specify the authentication mechanisms you wish to use.
   # This defaults to "pam" for PAM support, but may also include
   # "shadow" or "sasldb", like this:
   # MECHANISMS="pam shadow"

   MECHANISMS="pam"

   # Other options (default: -c)
   # See the saslauthd man page for information about these options.
   #
   # Example for postfix users: "-c -m /var/spool/postfix/var/run/saslauthd"
   # Note: See /usr/share/doc/sasl2-bin/README.Debian
   #OPTIONS="-c"

   #make sure you set the options here otherwise it ignores params above and will not work
   OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd"

.. note::

   If you prefer, you can use "shadow" instead of "pam". This will use MD5 hashed password transfer and is perfectly secure. The username and password needed to authenticate will be those of the users on the system you are using on the server.

Next, we update the dpkg "state" of /var/spool/postfix/var/run/saslauthd. The saslauthd init script uses this setting to create the missing directory with the appropriate permissions and ownership:

.. code-block:: ini

   dpkg-statoverride --force --update --add root sasl 755 /var/spool/postfix/var/run/saslauthd

This may report an error that "--update given" and the "/var/spool/postfix/var/run/saslauthd" directory does not exist. You can ignore this because when you start saslauthd next it will be created.

Finally, start saslauthd:

.. code-block:: sh

   sudo /etc/init.d/saslauthd start

测试
----

To see if SMTP-AUTH and TLS work properly now run the following command:

.. code-block:: sh

   telnet localhost 25

After you have established the connection to your postfix mail server type

.. code-block:: sh

   ehlo localhost

If you see the lines

.. code-block:: sh

   250-STARTTLS
   250-AUTH

among others, everything is working.

Type quit to return to the system's shell.

检修
----
 
从chroot移除Postfix 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If you run into issues while running Postfix you may be asked to remove Postfix from chroot to better diagnose the problem. 

为了做到这一点，你将需要编辑 /etc/postfix/master.cf 定位到以下行:

.. code-block:: ini

   smtp      inet  n       -       -       -       -       smtpd

并做以下修改:

.. code-block:: ini

   smtp      inet  n       -       n       -       -       smtpd

重启Postfix:

.. code-block:: sh

   sudo /etc/init.d/postfix restart

配置saslauthd为默认值
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you don't want to run Postfix in a chroot, or you'd like to not use chroot for troubleshooting purposes you will probably also want to return saslauthd back to its default configuration.

The first step in accomplishing this is to edit /etc/default/saslauthd comment the following lines we added above:

.. code-block:: ini

   #PWDIR="/var/spool/postfix/var/run/saslauthd"
   #PARAMS="-m ${PWDIR}"
   #PIDFILE="${PWDIR}/saslauthd.pid"

Then return the saslauthd dpkg "state" to its default location:


dpkg-statoverride --force --update --add root sasl 755 /var/run/saslauthd
And restart saslauthd:

.. code-block:: sh

   sudo /etc/init.d/saslauthd restart

为安全提交使用端口587
-------------------------------------------

If you want to use port 587 as the submission port for SMTP mail rather than 25 (many ISPs block port 25), you will need to edit /etc/postfix/master.cf and uncomment the line 

.. code-block:: ini

   submission inet n      -       n       -       -       smtpd

其它 Postfix 指南
--------------------------

These guides will teach you how to setup Postfix mail servers, from basic to advanced.

基本设置
^^^^^^^^^^^^^^^^

Postfix Basic Setup Howto will teach you the concepts of Posfix and how you can get Postfix basics set up and running. If you are new to Postfix it is recomended to follow this guide first.

虚拟邮箱和防病毒过滤
^^^^^^^^^^^^^^^^^^^^

Postfix Virtual MailBox ClamSmtp Howto will teach you how to setup virtual mailboxes using non-Linux accounts where each user will authenticate using their email address with Dovecot POP3/IMAP server and ClamSMTP Antivirus to filter both incoming and out going mails for known viruses.

发件人策略框架检测
^^^^^^^^^^^^^^^^^^^^

Postfix SPF will show you how to add SPF checking to your existing Postfix setup. This allows your server to reject mail from unauthorized sources.

设置DKIM电子邮件签名和验证
^^^^^^^^^^^^^^^^^^^^^^^^^^

Postfix DKIM will guide you through the setup process of dkim-milter for you existing Postfix installation. This will allow your server to sign and verify emails using DKIM.

添加Dspam
^^^^^^^^^^

Postfix Dspam will guide you through the setup process of dspam for you existing Postfix installation. This will enable on your mail server high quality statistical spam filter Dspam.

完整的解决方案
^^^^^^^^^^^^^^

Postfix Complete Virtual Mail System Howto will help you if you are managing a large number of virtual domains at an ISP level or in a large corporation where you mange few hundred or thousand mail domains. This guide is appropriate if you are looking a complete solution with:

#. Web based system administration
#. Unlimited number of domains
#. Virtual mail users without the need for shell accounts
#. Domain specific user names
#. Mailbox quotas
#. Web access to email accounts
#. Web based interface to change user passwords
#. IMAP and POP3 support
#. Auto responders
#. SMTP Authentication for secure relaying
#. SSL for transport layer security
#. Strong spam filtering
#. Anti-virus filtering
#. Log Analysis

Dovecot LDAP
^^^^^^^^^^^^^

The Postfix/DovecotLDAP guide will help you configure Postfix to use Dovecot as MDA with LDAP users. 

Dovecot SASL
^^^^^^^^^^^^^

The PostfixDovecotSASL guide will help you configure Postfix to use Dovecot's SASL implementation. Using Dovecot SASL may be preferable if you want to run Postfix in a chroot and need to use Cyrus SASL for other services.

.. note::

   this guide has been tested on Ubuntu 6.06 (Dapper) and Ubuntu 7.10 (Gutsy)


