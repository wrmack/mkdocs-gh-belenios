# Deploy to AWS

Assumptions:

- EC2 operating system is Amazon Linux 2023
- nginx is installed
- a subdomain for Belenios has been set up
- systemd-nspawn is installed
    - if not, install the systemd-container package: `sudo dnf install systemd-container`
- a belenios squashfs image has been generated as set out in [prepare an image](../install/preparation.md)

## Upload files to EC2
```bash
cd <belenios root>
rsync -av -e "ssh -i <path>MyEC2KeyPair.pem" contrib/fedora/belenios-nspawn ec2-user@<ip address>:/home/ec2-user
rsync -av -e "ssh -i <path>MyEC2KeyPair.pem" contrib/fedora/belenios-container@.service ec2-user@<ip address>:/home/ec2-user
rsync -av -e "ssh -i <path>MyEC2KeyPair.pem" _docker/build/belenios_3.1-4-g1b81705_arm64.squashfs  ec2-user@<ip address>:/home/ec2-user
rsync -av -e "ssh -i <path>MyEC2KeyPair.pem" demo/ocsigenserver.conf.in   ec2-user@<ip address>:/home/ec2-user
```

## Log in to AWS and setup belenios server as a service
```bash
# Log in
ssh -i <path>MyEC2KeyPair.pem ec2-user@<EC2 ip address>

# Copy belenios files to /srv 
sudo mkdir /srv/belenios-containers
sudo cp belenios-nspawn /srv/belenios-containers/
sudo mkdir /srv/belenios-containers/main
sudo cp belenios_3.1-4-g1b81705_arm64.squashfs /srv/belenios-containers/main
sudo ln -s /srv/belenios-containers/main/belenios_3.1-4-g1b81705_arm64.squashfs /srv/belenios-containers/main/rootfs.squashfs
sudo mkdir /srv/belenios-containers/main/belenios
sudo chown 1000:1000 /srv/belenios-containers/main/belenios
mkdir /srv/belenios-containers/main/belenios/etc
mkdir /srv/belenios-containers/main/belenios/var
cp ocsigenserver.conf.in /srv/belenios-containers/main/belenios/etc/

# Copy service spec to systemd configuration
sudo cp belenios-container@.service /etc/systemd/system

# Setup nginx (see below for configuration)
sudo nano /etc/nginx/sites-available/belenios
cd /etc/nginx/sites-enabled/
ln -s ../sites-available/belenios

# Run certbot to provide https for nginx - not covered here

# Adapt ocsigenserver.conf.in - see below for configuration
nano /srv/belenios-containers/main/belenios/etc/ocsigenserver.conf.in

# Make systemd aware of changes and start the Belenios service
sudo systemctl daemon-reload
sudo systemctl start belenios-container@main.service
```
## Configuration files

How my configuration files ended up.

### /etc/nginx/sites-available/belenios

```
server {
   root                      /var/www/html;
   server_name               belenios.wrmack.com;

   location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }

    listen [::]:443 ssl; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/belenios.wrmack.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/belenios.wrmack.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = belenios.wrmack.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


   listen                    80;
   listen                    [::]:80;
   server_name               belenios.wrmack.com;
    return 404; # managed by Certbot
}
```
### /srv/belenios-containers/main/belenios/etc/ocsigenserver.conf.in

```
<!-- -*- Mode: Xml -*- -->
<ocsigen>

  <server>

    <port>127.0.0.1:8001</port>

    <mimefile>_SHAREDIR_/mime.types</mimefile>

    <logdir>_VARDIR_/log</logdir>
    <datadir>_VARDIR_/lib</datadir>

    <uploaddir>_VARDIR_/upload</uploaddir>

    <!--
      The following limits are there to avoid flooding the server.
      <maxuploadfilesize> might need to be increased for handling large
      elections.
      <maxconnected> is related to the number of simultaneous voters
      visiting the server.
    -->
    <maxuploadfilesize>5120kB</maxuploadfilesize>
    <maxconnected>500</maxconnected>

    <commandpipe>_RUNDIR_/ocsigenserver_command</commandpipe>

    <charset>utf-8</charset>

    <extension name="staticmod"/>
    <extension name="redirectmod"/>

    <extension name="ocsipersist">
      <database file="_VARDIR_/lib/ocsidb"/>
    </extension>

    <extension name="eliom"/>

    <host charset="utf-8" hostfilter="*" defaulthostname="belenios.wrmack.com">
      <!-- <redirect suburl="^$" dest="http://www.example.org"/> -->
      <site path="static" charset="utf-8">
        <static dir="_SHAREDIR_/static" cache="0"/>
      </site>
      <eliom name="belenios">
        <public-url prefix="https://belenios.wrmack.com"/>
        <!-- Domain name used in Message-ID -->
        <domain name="belenios.wrmack.com"/>
        <!--
          The following can be adjusted to the capacity of your system.
          If <maxrequestbodysizeinmemory> is too small, large elections
          might fail, in particular with so-called alternative questions
          with many voters.
          <maxmailsatonce> depends heavily on how sending emails is
          handled by your system.
        -->
        <maxrequestbodysizeinmemory value="1048576"/>
        <maxmailsatonce value="1000"/>
        <tos uri="http://www.example.org/terms-of-service.html"/>
        <!-- <contact uri="mailto:contact@example.org"/> -->
        <server mail="elections@wrmack.com" return-path="elections@wrmack.com" name="Belenios public server"/>
        <auth-export name="builtin-password"/>
        <auth-export name="builtin-cas"/>
        <auth-export name="demo"><dummy/></auth-export> <!-- DEMO -->
        <auth-export name="email"><email/></auth-export> <!-- DEMO -->
        <auth name="demo"><dummy allowlist="demo_allowlist"/></auth> <!-- DEMO -->
        <auth name="local"><password db="local_passwords"/></auth> <!-- DEMO -->
        <auth name="public"><password db="public_passwords" allowsignups="true"/></auth>
        <auth name="email"><email/></auth> <!-- DEMO -->
        <auth name="captcha"><email use_captcha="true"/></auth> <!-- DEMO -->
        <!-- <auth name="google"><oidc server="https://accounts.google.com" client_id="client-id" client_secret="client-secret"/></auth> -->
        <source file="_SHAREDIR_/belenios.tar.gz"/>
        <logo file="_SHAREDIR_/static/placeholder.png" mime-type="image/png"/>
        <favicon file="_VARDIR_/favicon.ico" mime-type="image/png"/>
        <sealing file="demo/sealing.txt" mime-type="text/plain"/>
        <default-group group="Ed25519"/>
        <nh-group group="Ed25519"/>
        <share dir="_SHAREDIR_"/>
        <storage backend="filesystem">
          <uuid length="14"/>
          <spool dir="_VARDIR_/spool"/>
          <accounts dir="_VARDIR_/accounts"/>
          <map from="demo_allowlist" to="demo/dummy_logins.txt"/>
          <map from="local_passwords" to="demo/password_db.csv"/>
          <map from="public_passwords" to="_VARDIR_/password_db.csv"/>
        </storage>
        <admin-home file="_VARDIR_/admin-home.html"/>
        <success-snippet file="_VARDIR_/success-snippet.html"/>
        <warning file="_VARDIR_/warning.html"/>
        <footer file="_VARDIR_/footer.html"/>
        <!-- <deny-newelection/> -->
        <!--
            Uncomment the following line to disable revoting. Note that
            the ability to revote is important as a (light) measure
            against coercion.
        -->
        <!-- <deny-revote/> -->
      </eliom>
    </host>

  </server>

</ocsigen>
```


