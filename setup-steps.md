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


































