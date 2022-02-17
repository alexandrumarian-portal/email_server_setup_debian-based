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
