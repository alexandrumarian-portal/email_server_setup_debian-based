0. Install certbot nginx package
```
apt install certbot python3-certbot-nginx
certbot -d domain.ro -d mail.domain.ro
certbot -d www.domain.ro -d mail.domain.ro
#extra example
root@mail:~# sudo certbot --nginx --agree-tos --redirect --uir --hsts --staple-ocsp --must-staple -d domain.ro,webmail.domain.ro,www.domain.ro --email user@domain.ro
```
1. Installing Software Packages
```
apt update && apt install postfix postfix-mysql postfix-doc dovecot-common dovecot-imapd dovecot-pop3d libsasl2-2 libsasl2-modules libsasl2-modules-sql sasl2-bin libpam-mysql mailutils dovecot-mysql dovecot-sieve dovecot-managesieved
apt install mariadb-server mariadb-client
```
2. Configuring MySql and Connect it With Postfix
```
mysql_secure_installation 
....
mysql -u root 
mysql -u root -p 
mysql -u root
mysql> CREATE DATABASE mail;
mysql> USE mail;
mysql> CREATE USER 'mail_admin'@'localhost' IDENTIFIED BY 'TUTORIAL_PASSWORD';  
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON mail.* TO 'mail_admin'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> CREATE TABLE domains (domain varchar(50) NOT NULL, PRIMARY KEY (domain));
mysql> CREATE TABLE users (email varchar(80) NOT NULL, password varchar(128) NOT NULL, PRIMARY KEY (email));
mysql> CREATE TABLE forwardings (source varchar(80) NOT NULL, destination TEXT NOT NULL, PRIMARY KEY (source));
mysql> exit
```
3. Configuring Postfix to communicate with MySql
```
a) vim /etc/postfix/mysql_virtual_domains.cf 
user = mail_admin
password = TUTORIAL_PASSWORD
dbname = mail
query = SELECT domain FROM domains WHERE domain='%s'
hosts = 127.0.0.1

b) vim /etc/postfix/mysql_virtual_forwardings.cf
user = mail_admin
password = TUTORIAL_PASSWORD
dbname = mail
query = SELECT destination FROM forwardings WHERE source='%s'
hosts = 127.0.0.1

c) vim /etc/postfix/mysql_virtual_mailboxes.cf
user = mail_admin
password = TUTORIAL_PASSWORD
dbname = mail
query = SELECT CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/') FROM users WHERE email='%s'
hosts = 127.0.0.1

d) vim /etc/postfix/mysql_virtual_email2email.cf 
user = mail_admin
password = TUTORIAL_PASSWORD
dbname = mail
query = SELECT email FROM users WHERE email='%s'
hosts = 127.0.0.1

e) Setting the ownership and permissions
chmod o-rwx /etc/postfix/mysql_virtual_*
chown root.postfix /etc/postfix/mysql_virtual_*
```
4. Creating a user and group for mail handling
```
groupadd -g 5000 vmail
useradd -g vmail -u 5000 -d /var/vmail -m vmail
```
5. Configuring postfix
```
postconf -e "myhostname = mail.domain.ro"
postconf -e "mydestination = mail.domain.ro, localhost, localhost.localdomain"
postconf -e "mynetworks = 127.0.0.0/8"
postconf -e "message_size_limit = 31457280"
postconf -e "virtual_alias_domains ="
postconf -e "virtual_alias_maps = proxy:mysql:/etc/postfix/mysql_virtual_forwardings.cf, mysql:/etc/postfix/mysql_virtual_email2email.cf"
postconf -e "virtual_mailbox_domains = proxy:mysql:/etc/postfix/mysql_virtual_domains.cf"
postconf -e "virtual_mailbox_maps = proxy:mysql:/etc/postfix/mysql_virtual_mailboxes.cf"
postconf -e "virtual_mailbox_base = /var/vmail"
postconf -e "virtual_uid_maps = static:5000"
postconf -e "virtual_gid_maps = static:5000"
postconf -e "smtpd_sasl_auth_enable = yes"
postconf -e "broken_sasl_auth_clients = yes"
postconf -e "smtpd_sasl_authenticated_header = yes"
postconf -e "smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination"
postconf -e "smtpd_use_tls = yes"
postconf -e "smtpd_tls_cert_file = /etc/letsencrypt/live/domain.ro/fullchain.pem"
postconf -e "smtpd_tls_key_file = /etc/letsencrypt/live/domain.ro/privkey.pem"
postconf -e "virtual_transport=dovecot"
postconf -e 'proxy_read_maps = $local_recipient_maps $mydestination $virtual_alias_maps $virtual_alias_domains $virtual_mailbox_maps $virtual_mailbox_domains $relay_recipient_maps $relay_domains $canonical_maps $sender_canonical_maps $recipient_canonical_maps $relocated_maps $transport_maps $mynetworks $virtual_mailbox_limit_maps'


vi /etc/postfix/master.cf  - paste : 

submission     inet     n    -    y    -    -    smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_tls_wrappermode=no
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o smtpd_recipient_restrictions=permit_mynetworks,permit_sasl_authenticated,reject
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
  
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o smtpd_recipient_restrictions=permit_mynetworks,permit_sasl_authenticated,reject
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth  
  
```
6. Configuring SMTP AUTH (SASLAUTHD and MySql)
```
a) Creating a directory where saslauthd will save its information:  
mkdir -p /var/spool/postfix/var/run/saslauthd

b) Editing the configuration file of saslauthd: vim /etc/default/saslauthd
START=yes
DESC="SASL Authentication Daemon"
NAME="saslauthd"
MECHANISMS="pam"
MECH_OPTIONS=""
THREADS=5
OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd -r"

c) Creating a new file: vim /etc/pam.d/smtp
auth required pam_mysql.so user=mail_admin passwd=TUTORIAL_PASSWORD host=127.0.0.1 db=mail table=users usercolumn=email passwdcolumn=password crypt=3
account sufficient pam_mysql.so user=mail_admin passwd=TUTORIAL_PASSWORD host=127.0.0.1 db=mail table=users usercolumn=email passwdcolumn=password crypt=3

d) vim /etc/postfix/sasl/smtpd.conf
pwcheck_method: saslauthd 
mech_list: plain login 
log_level: 4

e) Setting the permissions
chmod o-rwx /etc/pam.d/smtp
chmod o-rwx /etc/postfix/sasl/smtpd.conf

f) Adding the postfix user to the sasl group for group access permissions: 
usermod  -aG sasl postfix

g) Restarting the services:
systemctl restart postfix
systemctl restart saslauthd
```
7. Configuring Dovecot (POP3/IMAP)
```
a) At the end of /etc/postfix/master.cf add:
dovecot   unix  -       n       n       -       -       pipe
    flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -d ${recipient}

b) Edit Dovecot config file: vim /etc/dovecot/dovecot.conf
log_timestamp = "%Y-%m-%d %H:%M:%S "
mail_location = maildir:/var/vmail/%d/%n/Maildir
managesieve_notify_capability = mailto
managesieve_sieve_capability = fileinto reject envelope encoded-character vacation subaddress comparator-i;ascii-numeric relational regex imap4flags copy include variables body enotify environment mailbox date
namespace {
  inbox = yes
  location = 
  prefix = INBOX.
  separator = .
  type = private
}
passdb {
  args = /etc/dovecot/dovecot-sql.conf
  driver = sql
}
protocols = imap pop3

service auth {
  unix_listener /var/spool/postfix/private/auth {
    group = postfix
    mode = 0660
    user = postfix
  }
  unix_listener auth-master {
    mode = 0600
    user = vmail
  }
  user = root
}

userdb {
  args = uid=5000 gid=5000 home=/var/vmail/%d/%n allow_all_users=yes
  driver = static
}

protocol lda {
  auth_socket_path = /var/run/dovecot/auth-master
  log_path = /var/vmail/dovecot-deliver.log
  mail_plugins = sieve
  postmaster_address = postmaster@example.com
}

protocol pop3 {
  pop3_uidl_format = %08Xu%08Xv
}

service stats {
  unix_listener stats-reader {
    user = dovecot
    group = vmail
    mode = 0660
  }
  unix_listener stats-writer {
    user = dovecot
    group = vmail
    mode = 0660
  }
}

ssl = yes
ssl_cert = </etc/letsencrypt/live/domain.ro/fullchain.pem
ssl_key = </etc/letsencrypt/live/domain.ro/privkey.pem

c) vim /etc/dovecot/dovecot-sql.conf
driver = mysql
connect = host=127.0.0.1 dbname=mail user=mail_admin password=TUTORIAL_PASSWORD
default_pass_scheme = PLAIN-MD5
password_query = SELECT email as user, password FROM users WHERE email='%u';

d) Restart Dovecot
systemctl restart dovecot
```
8. Adding Domains and Virtual Users. 
```
mysql -u root
msyql>USE mail;
mysql>INSERT INTO domains (domain) VALUES ('domain.ro');
mysql>insert into users(email,password) values('marian@domain.ro', md5('PASSWORD'));
mysql>insert into users(email,password) values('me@domain.ro', md5('PASSWORD'));
mysql>quit;
```
9. Check mail domain, account, alias 
```
root@domain:/etc/postfix# postmap -q domain.ro mysql:/etc/postfix/mysql_virtual_domains.cf
domain.ro
root@domain:/etc/postfix# postmap -q marian@domain.ro mysql:/etc/postfix/mysql_virtual_mailboxes.cf
domain.ro/marian/
root@domain:/etc/postfix# postmap -q me@domain.ro mysql:/etc/postfix/mysql_virtual_mailboxes.cf
domain.ro/me/
root@domain:/etc/postfix# 

#postfconf new/actual value
root@mail:/etc/postfix# postconf -n message_size_limit
message_size_limit = 31457280

#postconf default value 
root@mail:/etc/postfix# postconf -d message_size_limit
message_size_limit = 10240000
root@mail:/etc/postfix#  
  
#check authentication 
root@mail:~# testsaslauthd -u user@domain.ro -p PAROLA -f /var/spool/postfix/var/run/saslauthd/mux -s smtp
0: OK "Success."
root@mail:~#
root@mail:~# testsaslauthd -u marian@domain.ro -p BAD_PAROLA -f /var/spool/postfix/var/run/saslauthd/mux -s smtp
0: NO "authentication failed"

#enable 465 smtp port in /etc/postfix/master.cf
smtps     inet  n		-		y		-		-		smtpd

#Enable port 587 in postfix vi /etc/postfix/master.cf
submission inet n - n - - smtpd

#increase the smtp logging by add -v in /etc/postfix/master.cf
smtp      inet  n       -       y       -       -       smtpd  -v


systemctl restart postfix
```

#Amavis Installation Guide
#All commands are run as root 
```
1. Installing Amavis
apt update && apt install amavisd-new
 
Note: if there's an error set $myhostname in /etc/amavis/conf.d/05-node_id
 
2. Installing required packages for scanning attachments
apt install arj bzip2 cabextract cpio rpm2cpio file gzip lhasa nomarch pax rar unrar p7zip-full unzip zip lrzip lzip liblz4-tool lzop unrar-free
 
3. Configuring Postfix (/etc/postfix/main.cf)
postconf -e 'content_filter = smtp-amavis:[127.0.0.1]:10024'
postconf -e 'smtpd_proxy_options = speed_adjust'
 
4. Add to the end of /etc/postfix/master.cf
smtp-amavis   unix   -   -   n   -   2   smtp
    -o syslog_name=postfix/amavis
    -o smtp_data_done_timeout=1200
    -o smtp_send_xforward_command=yes
    -o disable_dns_lookups=yes
    -o max_use=20
    -o smtp_tls_security_level=none
 
 
127.0.0.1:10025   inet   n    -     n     -     -    smtpd
    -o syslog_name=postfix/10025
    -o content_filter=
    -o mynetworks_style=host
    -o mynetworks=127.0.0.0/8
    -o local_recipient_maps=
    -o relay_recipient_maps=
    -o strict_rfc821_envelopes=yes
    -o smtp_tls_security_level=none
    -o smtpd_tls_security_level=none
    -o smtpd_restriction_classes=
    -o smtpd_delay_reject=no
    -o smtpd_client_restrictions=permit_mynetworks,reject
    -o smtpd_helo_restrictions=
    -o smtpd_sender_restrictions=
    -o smtpd_recipient_restrictions=permit_mynetworks,reject
    -o smtpd_end_of_data_restrictions=
    -o smtpd_error_sleep_time=0
    -o smtpd_soft_error_limit=1001
    -o smtpd_hard_error_limit=1000
    -o smtpd_client_connection_count_limit=0
    -o smtpd_client_connection_rate_limit=0
    -o receive_override_options=no_header_body_checks,no_unknown_recipient_checks,no_address_mappings
 
5. Installing ClamAV
apt install clamav clamav-daemon
 
6. Turning on virus-checking in Amavis.
In /etc/amavis/conf.d/15-content_filter_mode
 
Uncomment:
@bypass_virus_checks_maps = (
  	\%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);
 
7. Restarting Amavis and ClamAv
systemctl restart amavis; systemctl restart clamav-daemon




vim /etc/postfix/main.cf

smtpd_helo_required = yes
smtpd_helo_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_invalid_helo_hostname,
  reject_non_fqdn_helo_hostname,
  reject_unknown_helo_hostname,
  permit

smtpd_sender_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_unknown_sender_domain,
  reject_non_fqdn_sender,
  reject_unknown_reverse_client_hostname,
  #could trigger false-positives
  reject_unknown_client_hostname,  
  permit

smtpd_recipient_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_unauth_destination,
  reject_unauth_pipelining,
  reject_unknown_recipient_domain,
  reject_non_fqdn_recipient,
  check_client_access hash:/etc/postfix/rbl_override,
  reject_rhsbl_helo dbl.spamhaus.org,
  reject_rhsbl_reverse_client dbl.spamhaus.org,
  reject_rhsbl_sender dbl.spamhaus.org,
  reject_rbl_client zen.spamhaus.org,
  permit

/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\
/!\/!\/!\/!\/!\ comment old smtpd_recipient_restrictions entry /!\/!\/!\/!\
/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\

create a whitelist of domains :
vim /etc/postfix/rbl_override 
domain1.com OK
gooddomain.com OK

postmap /etc/postfix/rbl_override 
```


#Rspamd Installation Guide
#All commands are run as root 
```
1. Installing Redis as storage for non-volatile data and as a cache for volatile data
apt update && apt install redis-server
 
2. Adding the repository GPG key 
wget -O- https://rspamd.com/apt-stable/gpg.key | sudo apt-key add -
 
3. Enabling the Rspamd repository
echo "deb http://rspamd.com/apt-stable/ $(lsb_release -cs) main" | sudo tee -a /etc/apt/sources.list.d/rspamd.list
 
4. Installing Rspamd
apt update && apt install rspamd
 
5. Configuring the Rspamd normal worker to listen only on localhost interface
vim /etc/rspamd/local.d/worker-normal.inc
    bind_socket = "127.0.0.1:11333";
 
6. Enabling the milter protocol to communicate with postfix:
vim /etc/rspamd/local.d/worker-proxy.inc
    bind_socket = "127.0.0.1:11332";
    milter = yes;
    timeout = 120s;
    upstream "local" {
    default = yes;
    self_scan = yes;
    }
 
7. Configure postfix to use Rspamd
postconf -e "milter_protocol = 6"
postconf -e "milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}"
postconf -e "milter_default_action = accept"
postconf -e "smtpd_milters = inet:127.0.0.1:11332"
postconf -e "non_smtpd_milters = inet:127.0.0.1:11332"
 
8. Restarting Rspamd and Postfix
systemctl restart rspamd; systemctl restart postfix

```
#Install and setup Roundcube webmail :
```
wget https://github.com/roundcube/roundcubemail/releases/download/1.4.11/roundcubemail-1.4.11-complete.tar.gz

tar xvf roundcubemail*
sudo mv roundcubemail* /var/www/roundcube

sudo apt install php-net-ldap2 php-net-ldap3 php-imagick php7.4-common php7.4-gd php7.4-imap php7.4-json php7.4-curl php7.4-zip php7.4-xml php7.4-mbstring php7.4-bz2 php7.4-intl php7.4-gmp
sudo apt install php-fpm php-mysql
sudo apt install composer
cd /var/www/roundcube
composer install --no-dev
sudo chown www-data:www-data temp/ logs/ -R



root@mail:/var/www/roundcube# sudo mysql -u root
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 105
Server version: 10.3.31-MariaDB-0ubuntu0.20.04.1 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE roundcube DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]>  CREATE USER roundcubeuser@localhost IDENTIFIED BY 'TUTORIAL_PASSWORD';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON roundcube.* TO roundcubeuser@localhost;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> quit
Bye
root@mail:/var/www/roundcube#


sudo mysql roundcube < /var/www/roundcube/SQL/mysql.initial.sql
or 
sudo mysql roundcube -u roundcubeuser -p < /var/www/html/roundcube/SQL/mysql.initial.sql


Set timezone in php ini file :
vim /etc/php/7.4/fpm/php.ini 


sudo nano /etc/nginx/conf.d/roundcube.conf
server {
  listen 80;
  listen [::]:80;
  server_name mail.example.com;
  root /var/www/roundcube/;
  index index.php index.html index.htm;

  error_log /var/log/nginx/roundcube.error;
  access_log /var/log/nginx/roundcube.access;

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/mail.marianc.gq/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/mail.marianc.gq/privkey.pem; # managed by Certbot

  location / {
    try_files $uri $uri/ /index.php;
  }

  location ~ \.php$ {
   try_files $uri =404;
    fastcgi_pass unix:/run/php/php7.4-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
  }

  location ~ /.well-known/acme-challenge {
    allow all;
  }
 location ~ ^/(README|INSTALL|LICENSE|CHANGELOG|UPGRADING)$ {
    deny all;
  }
  location ~ ^/(bin|SQL)/ {
    deny all;
  }
 # A long browser cache lifetime can speed up repeat visits to your page
  location ~* \.(jpg|jpeg|gif|png|webp|svg|woff|woff2|ttf|css|js|ico|xml)$ {
       access_log        off;
       log_not_found     off;
       expires           360d;
  }
}

sudo nginx -t
sudo systemctl reload nginx

(you can add "return 301 https://$host$request_uri;" in the nginx config file for implemmenting http2https redirects )


The IMAP and SMTP section allows you to configure how to receive and submit email. Enter the following values for IMAP.

IMAP host: ssl://mail.example.com port: 993
SMTP port: tls://mail.example.com port: 587. Note that you must use tls:// as the prefix for port 587. The ssl:// prefix should be used for port 465.

you can scroll down to the Plugins section to enable some plugins. For example, the password plugin, mark as junk plugin, and so on. I enabled all of them. 
(You can always disable a plugin after installation in the /var/www/roundcube/config/config.inc.php file.)


Once that???s done, click create config button which will create configuration based on the information you entered. 
You need to copy the configuration and save it as config.inc.php under the /var/www/roundcube/config/ directory.
Once the config.inc.php file is created, click continue button. In the final step, test your SMTP and IMAP settings 
by sending a test email and checking IMAP login. Note that you need to enter your full email address in the Sender field when testing SMTP config.


How to Upgrade Roundcube
wget https://github.com/roundcube/roundcubemail/releases/download/1.5.0/roundcubemail-1.5.0-complete.tar.gz
tar xvf roundcubemail-1.5.0-complete.tar.gz
chown www-data:www-data roundcubemail-1.5.0/ -R
roundcubemail-1.5.0/bin/installto.sh /var/www/roundcube/



sudo apt install dovecot-sieve dovecot-managesieved
sudo apt install dovecot-lmtpd

sudo nano /etc/dovecot/dovecot.conf
Add lmtp and sieve to the supported protocols.
protocols = imap lmtp sieve

sudo nano /etc/dovecot/conf.d/10-master.conf
service lmtp {
 unix_listener /var/spool/postfix/private/dovecot-lmtp {
   group = postfix
   mode = 0600
   user = postfix
  }
}


Edit the Dovecot main configuration file.
sudo nano /etc/dovecot/dovecot.conf
Add lmtp and sieve to the supported protocols:
protocols = imap lmtp sieve

Then edit the Dovecot 10-master.conf file.
sudo nano /etc/dovecot/conf.d/10-master.conf
Change the lmtp service definition to the following.
service lmtp {
 unix_listener /var/spool/postfix/private/dovecot-lmtp {
   group = postfix
   mode = 0600
   user = postfix
  }
}

Next, edit the Postfix main configuration file.
sudo nano /etc/postfix/main.cf
Add the following lines at the end of the file. The first line tells Postfix to deliver emails to local message store via the dovecot LMTP server. The second line disables SMTPUTF8 in Postfix, because Dovecot-LMTP doesn???t support this email extension:

mailbox_transport = lmtp:unix:private/dovecot-lmtp
smtputf8_enable = no

sudo systemctl restart postfix dovecot

Configure cron jobs for renewal of certificate and for expunge folders :

crontab -e

MAILTO="USER@DOMAIN.COM"
@daily certbot renew --quiet && systemctl reload postfix dovecot nginx
@daily doveadm expunge -A mailbox Trash savedbefore 2w && doveadm expunge -A mailbox Junk savedbefore 2w && doveadm expunge -A mailbox Sent savedbefore 2w





sudo nano /etc/postfix/main.cf
Add the following lines at the end of the file
mailbox_transport = lmtp:unix:private/dovecot-lmtp
smtputf8_enable = no


sudo nano /etc/dovecot/conf.d/15-lda.conf

protocol lda {
    # Space separated list of plugins to load (default is global mail_plugins).
    mail_plugins = $mail_plugins sieve
}


sudo systemctl restart postfix dovecot

```
 
#Configure SPF, DKIM and DMARC records :
```
    SPF to tell the world which addresses have the right send your emails
    MX to tell the world which addresses will receive the emails and in which order
    DKIM (a public key) to allow recipients to check your emails really comes from your servers (signed used a private key)
    DMARC to tell recipient what to do with mails not respecting SPF
```
```
apt install postfix-policyd-spf-python


vim /etc/postfix/master.cf and add at the end the following lines: 
policyd-spf  unix  -       n       n       -       0       spawn
    user=policyd-spf argv=/usr/bin/policyd-spf
    
    
    
vim /etc/postfix/main.cf   and edit the config file as :

policyd-spf_time_limit = 3600
smtpd_recipient_restrictions =
   permit_mynetworks,
   permit_sasl_authenticated,
   reject_unauth_destination,
   check_policy_service unix:private/policyd-spf

    
in the forward bind file, add a TXT record (/etc/bind/forward.domain.com ex.) and increase the bind serial :) :
marianc.gq.        IN      TXT     "v=spf1 a mx -all"
_dmarc    IN       TXT     "v=DMARC1; p=none; pct=100; fo=1; rua=mailto:USER@DOMAIN.COM"
(please expect to receive emails from google,yahoo regarding your status)

alernatively, you can use the following if you do not want reports :
@	IN	TXT		"v=spf1 a mx ip4:xxx.xxx.xxx.xxx -all"
_dmarc	IN	TXT		"v=DMARC1; p=quarantine; pct=100"




service bind9 restart
service postfix restart


Installing and Configuring DKIM

apt update && sudo apt install opendkim opendkim-tools
systemctl enable opendkim

gpasswd -a postfix opendkim

vim /etc/opendkim.conf and Uncomment or add lines as below:

Canonicalization        relaxed/simple
Mode                    sv
SubDomains              no

UMask                   007

AutoRestart             yes
AutoRestartRate         10/1M
Background              yes
DNSTimeout              5
SignatureAlgorithm      rsa-sha256

#OpenDKIM user
# Remember to add user postfix to group opendkim
UserID                  opendkim

# Map domains in From addresses to keys used to sign messages
KeyTable                refile:/etc/opendkim/key.table
SigningTable            refile:/etc/opendkim/signing.table

# Hosts to ignore when verifying signatures
ExternalIgnoreList      /etc/opendkim/trusted.hosts
InternalHosts           /etc/opendkim/trusted.hosts



mkdir /etc/opendkim
mkdir /etc/opendkim/keys
chown -R opendkim:opendkim /etc/opendkim
chmod go-rw /etc/opendkim/keys


vim /etc/opendkim/signing.table
*@example.com default._domainkey.example.com

nano /etc/opendkim/key.table
default._domainkey.example.com    example.com:default:/etc/opendkim/keys/example.com/default.private


vim /etc/opendkim/trusted.hosts
127.0.0.1
localhost

hostname
example.com
*.example.com


mkdir /etc/opendkim/keys/example.com
opendkim-genkey -b 2048 -d example.com -D /etc/opendkim/keys/example.com -s default -v


Display the public key:
cat /etc/opendkim/keys/example.com/default.txt


default._domainkey	IN	TXT	( "v=DKIM1; h=sha256; k=rsa; "
	  "p=MIIBIjANBgkqhkiG9xxxxxxxxxdqlBZW527Abv/SAIVP0mE8ZiDdcn6PjrVF48SFxxxxxxxxxxxRtFmq8D1xxxxxxxxxxxxxp96rfX"
	  "L8YXBCxxxxxxxxxxxxxxxxxxGQEyt3MQVegxxxxxxxxxxxxxxQhEcQIDAQAB" )  ; ----- DKIM key default for domain.com

and add the record to the forward zone config file of bind dns server and increase the serial :) 



opendkim-testkey -d example.com -s default -vvv
If you see Key not secure in the command output, don???t panic. This is because DNSSEC isn???t enabled on your domain name. 
DNSSEC is a security standard for secure DNS query. Most domain names haven???t enabled DNSSEC. 
There???s absolutely no need to worry about Key not secure. You can continue to follow this guide.


Connect Postfix to OpenDKIM
sudo mkdir /var/spool/postfix/opendkim
sudo chown opendkim:postfix /var/spool/postfix/opendkim


sudo nano /etc/opendkim.conf
Find the following line:
Socket    local:/run/opendkim/opendkim.sock

Replace it with the following line:
Socket    local:/var/spool/postfix/opendkim/opendkim.sock


sudo vim /etc/default/opendkim 
Find the following line.
SOCKET=local:$RUNDIR/opendkim.sock

Change it to:
SOCKET="local:/var/spool/postfix/opendkim/opendkim.sock"


sudo nano /etc/postfix/main.cf
Add the following lines at the end of this file, so Postfix will be able to call OpenDKIM via the milter protocol.

# Milter configuration
milter_default_action = accept
milter_protocol = 6
smtpd_milters = local:opendkim/opendkim.sock
non_smtpd_milters = $smtpd_milters

sudo systemctl restart opendkim postfix

```

Useful bash scripts :


```
cat << 'EOF' > /usr/local/bin/add_domain
#!/bin/bash
usage () {
    echo "Usage: add_domain <domain_name>"
}
 
if [ $# -ne 1 ] ; then
    usage
    exit 1
fi
 
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
 
/usr/bin/mysql -u postfix db_postfix $OPTION -e "INSERT INTO db_postfix.virtual_domains (name) VALUES ('$1');"
EOF
```
```
cat << 'EOF' > /usr/local/bin/add_user
#!/bin/bash
if [ $# -ne 2 ] ; then
    echo "Usage: add_user <e-mail> <password>"
    exit 1
fi
 
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
regex="^[a-z0-9!#\$%&'*+/=?^_\`{|}~-]+(\.[a-z0-9!#$%&'*+/=?^_\`{|}~-]+)*@([a-z0-9]([a-z0-9-]*[a-z0-9])?\.)+[a-z0-9]([a-z0-9-]*[a-z0-9])?\$"
 
if [[ $1 =~ $regex ]] ; then
    DOMAIN=`echo $1 | cut -d'@' -f2`
else
    echo "Not a valid e-mail address."
    exit 1
fi
ID=$(/usr/bin/mysql -u postfix db_postfix -s $OPTION -N -e "SELECT id from db_postfix.virtual_domains WHERE NAME='$DOMAIN';")
if [ -z "$ID" ] ; then
    echo "The domain $DOMAIN is missing. Use add_domain $DOMAIN first."
else
   /usr/bin/mysql -u postfix db_postfix $OPTION -e "INSERT INTO db_postfix.virtual_users (domain_id, email, password) VALUES ($ID, '$1', ENCRYPT('$2', CONCAT('\$6\$', SUBSTRING(SHA(RAND()), -16))));"
fi
EOF
```
```
cat << 'EOF' > /usr/local/bin/ls_domains
#!/bin/bash
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
/usr/bin/mysql -u postfix db_postfix $OPTION -e "SELECT * FROM virtual_domains;"
EOF
```
```
cat << 'EOF' > /usr/local/bin/ls_users
#!/bin/bash
 
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
 
/usr/bin/mysql -u postfix db_postfix $OPTION -e "SELECT * FROM virtual_users;"
 
EOF
```
```
cat << 'EOF' > /usr/local/bin/rmall_domains
#!/bin/bash
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
 
while true; do
    read -p "All domain records will be deleted. Please confirm [yn] " yn
    case $yn in
        [Yy]* ) /usr/bin/mysql -u postfix db_postfix $OPTION -e "DELETE FROM db_postfix.virtual_domains"; break;;
        [Nn]* ) exit;;
        * ) echo "Answer yes(y) or no(n).";;
    esac
done
EOF
```
```
cat << 'EOF' > /usr/local/bin/rmall_users
#!/bin/bash
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
 
while true; do
    read -p "All user records will be deleted. Please confirm [yn] " yn
    case $yn in
        [Yy]* ) /usr/bin/mysql -u postfix db_postfix $OPTION -e "DELETE FROM db_postfix.virtual_users"; break;;
        [Nn]* ) exit;;
        * ) echo "Answer yes(y) or no(n).";;
    esac
done
 
EOF
```
```
cat << 'EOF' > /usr/local/bin/rm_domain
#!/bin/bash
if [ $# -ne 1 ]
  then
    echo "Usage: rm_domain <domain_name>"
    exit
fi
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
 
/usr/bin/mysql -u postfix db_postfix $OPTION -e "DELETE FROM db_postfix.virtual_domains WHERE name = '$1';"
 
EOF
```
```
cat << 'EOF' > /usr/local/bin/rm_user
#!/bin/bash
if [ $# -ne 1 ]
  then
    echo "Usage: rm_user <e-mail>"
    exit
fi
if [ -z "$MYSQL_PWD" ] ; then
    OPTION="-p"
else
    OPTION=""
fi
 
/usr/bin/mysql -u postfix db_postfix $OPTION -e "DELETE FROM db_postfix.virtual_users WHERE email = '$1';"
EOF
```
```
cd /usr/local/bin
chmod +x add_domain ls_domains rmall_domains rm_domain add_user ls_users rmall_users rm_user
export MYSQL_PWD=pwd_for_postfix_user
add_domain cloudranger.live
add_user klimenta@cloudranger.live some_pwd

```


