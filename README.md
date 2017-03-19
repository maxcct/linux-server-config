# linux-server-config
Account of the process of configuring a Linux server to serve my NGO Emporium app.

## IP Address
`34.207.99.196`
## URL
`http://ec2-34-207-99-196.compute-1.amazonaws.com/`


## Software installed
Everything installed by `sudo apt-get update` + `sudo apt-get upgrade`.
Other installed software includes: apache2, flask, git, mod-wsgi, postgresql, oauth2client, sqlalchemy, httplib2, requests, flask-seasurf.


## Configurations
### Firewall
Add `custom TCP 2200` to 'Firewall' section of 'Networking' tab for server instance in Lightsail.

Set defaults to deny all incoming connections and allow all outgoing:
`sudo ufw default deny incoming`
`sudo ufw default allow outgoing`

Allow incoming connections on port 2200 (for SSH), port 80 (for HTTP) and port 123 (for NTP):
`sudo ufw allow 2200/tcp`
`sudo ufw allow www`
`sudo ufw allow ntp`

Enable firewall:
`sudo ufw enable`

### SSH port

In the file `/etc/ssh/sshd_config`, I change `Port 22` to `Port 2200`, then used `sudo service ssh restart`.

### postgreSQL
Created 'catalog' database user: `sudo -u postgres createuser -P catalog`
Created database: `sudo -u postgres createdb -O catalog catalog`
Set up catalog user with:
`ALTER ROLE catalog WITH PASSWORD 'catalog';`
`ALTER USER catalog CREATEDB;`
`ALTER DATABASE catalog OWNER TO catalog;`
Used `\c catalog` to enter database, then:
`REVOKE ALL ON SCHEMA public FROM public;`
`GRANT ALL ON SCHEMA public TO catalog;'`

### Not needed
I did not need to disable root login or change the time zone, as these were already at the correct settings.


## Setting up app
I cloned my app repository into `/var/www`, and rearranged the directory structure as required.

I renamed `project.py` to `__init__.py` and changed `engine = create_engine('sqlite:///ngosandusers.db')` to `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`. I also made this change in `database_setup.py`. I then removed all `functools` fuctionality from the app, as `functools` appears to be impossible to install properly on an Ubuntu server (I managed to install it once, but then pip broke completely, and I had to scrap the whole instance).

I then changed the relevant fields in `g_client_secrets.json` and on the Google Cloud Platform to allow my server's URL.

Next, I created the file `ngoemporium.wsgi` in the top level of my app's directories, with the following contents:

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/NGO_Emporium/")

from NGO_Emporium import app as application
application.secret_key = 'super_secret_key'
```

Then in `/etc/apache2/sites-available$` I created the file `NGO_Emporium.conf` with the following contents:

```
<VirtualHost *:80>
                ServerName 34.207.99.196
                ServerAdmin admin@mywebsite.com
                WSGIScriptAlias / /var/www/NGO_Emporium/ngoemporium.wsgi
                <Directory /var/www/NGO_Emporium/NGO_Emporium>
                        Require all granted
                </Directory>
                Alias /static /var/www/NGO_Emporium/NGO_Emporium/static
                <Directory /var/www/NGO_Emporium/NGO_Emporium/static/>
                        Require all granted
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

I followed [the instructions here](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps) to navigate through the virtual environment process. After a bunch of troubleshooting, the site was then live.

##Resources
https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu-14-04
https://docs.google.com/document/d/1pvE6os2ctLevO_EBmg3Leq4VEc1DbORcxQ_zpPqVr78/edit
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
http://docs.sqlalchemy.org/en/latest/core/engines.html
https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
