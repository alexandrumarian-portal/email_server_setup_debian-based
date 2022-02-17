```
apt install certbot python3-certbot-nginx
certbot -d domain.ro -d mail.domain.ro
certbot -d www.domain.ro -d mail.domain.ro
#extra example
root@mail:~# sudo certbot --nginx --agree-tos --redirect --uir --hsts --staple-ocsp --must-staple -d domain.ro,webmail.domain.ro,www.domain.ro --email user@domain.ro
```
