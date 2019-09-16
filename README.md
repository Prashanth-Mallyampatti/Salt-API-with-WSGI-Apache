# Salt-API-with-WSGI-Apache-

Steps to configure Apache and Salt API for minion test pings:

Note: Unix system used to develop and test the following username and password: saltapi and saltapi

1) ```
   apt-get update
   apt-get upgrade
   apt-get install python-pip
   pip install pyopenssl
   apt-get update
   add-apt-repository ppa:deadsnakes/ppa
   apt-get install python3.7
   ```

2) Install Salt Master, Minion, API
   ```
   wget -O - https://repo.saltstack.com/apt/ubuntu/18.04/amd64/latest/SALTSTACK-GPG-KEY.pub | sudo apt-key add -

   Add the following line to /etc/apt/sources.list.d/saltstack.list 

   deb http://repo.saltstack.com/apt/ubuntu/18.04/amd64/latest bionic main
   
   apt-get update
   apt-get install salt-master
   apt-get install salt-minion
   apt-get install salt-api
   ```

3) Make sure you have the latest Salt installed. Salt API with Apache has been tested out on 2019.2.0 salt version
```
   root@saltapi:~# salt --versions-report
Salt Version:
           Salt: 2019.2.0

Dependency Versions:
           cffi: Not Installed
       cherrypy: unknown
       dateutil: 2.6.1
      docker-py: Not Installed
          gitdb: 2.0.3
      gitpython: 2.1.8
          ioflo: Not Installed
         Jinja2: 2.10
        libgit2: Not Installed
        libnacl: Not Installed
       M2Crypto: Not Installed
           Mako: 1.0.7
   msgpack-pure: Not Installed
 msgpack-python: 0.5.6
   mysql-python: Not Installed
      pycparser: Not Installed
       pycrypto: 2.6.1
   pycryptodome: Not Installed
         pygit2: Not Installed
         Python: 2.7.15+ (default, Jul  9 2019, 16:51:35)
   python-gnupg: 0.4.1
         PyYAML: 3.12
          PyZMQ: 16.0.2
           RAET: Not Installed
          smmap: 2.0.3
        timelib: Not Installed
        Tornado: 4.5.3
            ZMQ: 4.2.5

System Versions:
           dist: Ubuntu 18.04 bionic
         locale: UTF-8
        machine: x86_64
        release: 4.15.0-62-generic
         system: Linux
        version: Ubuntu 18.04 bionic
```

4) Install CherryPy
```
pip install -U Cherrypy
apt-get install libapache2-mod-wsgi
```
There's a known issue with Salt API and cherrypy regarding SSL certs: https://github.com/cherrypy/cherrypy/issues/1298

Apparently [Cherrypy 3.2.3](https://bitbucket.org/cherrypy/cherrypy/downloads/) seems to solve the problem but had no luck when actually tried out. 

To check cherrypy version: 
```
 python -c "import cherrypy;print cherrypy.__version__"
```

5) Install Apache2:
```
apt install apache2
```

6) Configure Salt API:

Generate a self-signed certificate using the create_self_signed_cert() execution function.
```
root@saltapi:~# salt-call --local tls.create_self_signed_cert
local:
    Created Private Key: "/etc/pki/tls/certs/localhost.key." Created Certificate: "/etc/pki/tls/certs/localhost.crt."

```
```
root@saltapi:~# cat /etc/salt/master.d/salt-api.conf
rest_cherrypy:
  port: 8000
  disable_ssl: False
  ssl_crt: /etc/pki/tls/certs/localhost.crt
  ssl_key: /etc/pki/tls/certs/localhost.key

external_auth:
 pam:
  saltapi:
   - .*
```
Uncomment the following line in /etc/salt/master:
```
default_include: master.d/*.conf
```
Restart services
```
 systemctl restart salt-master
 systemctl restart salt-api
```

7) Configure Apache

Edit and add the following lines to: /etc/apache2/apache2.conf
```
LogLevel warn

# Include module configuration:
IncludeOptional mods-enabled/*.load
IncludeOptional mods-enabled/*.conf

# Include list of ports to listen on
Include ports.conf

LoadModule alias_module /usr/lib/apache2/modules/mod_alias.so

# Sets the default security model of the Apache2 HTTPD server. It does
# not allow access to the root filesystem outside of /usr/share and /var/www.
# The former is used by web applications packaged in Debian,
# the latter may be used for local directories served by the web server. If
# your system is serving content from a sub-directory in /srv you must allow
# access here, or in any related virtual host.
<Directory />
        Options FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>

<Directory /usr/share>
        AllowOverride None
        Require all granted
</Directory>

<Directory /var/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>

<Directory /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy/>
        AllowOverride None
        Options Indexes FollowSymLinks
        Require all granted
        Allow from all
</Directory>
```
Add mod_alias.so as shown below and above obtained from: find / -name "mod_alias.so"
```
LoadModule alias_module /usr/lib/apache2/modules/mod_alias.so
```

Configure virtual host: 

Here port 80 has been configured. Please make sure to add "Listen <port_number>" in /etc/apache2/ports.conf if for any other port. 
```
root@saltapi:~# cat /etc/apache2/sites-enabled/000-default.conf
<VirtualHost *:80>
       ServerAdmin webmaster@localhost
       DocumentRoot /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy
       ErrorLog /var/log/apache2/error.log
       CustomLog /var/log/apache2/access.log combined

LogLevel debug
LoadModule alias_module /usr/lib/apache2/modules/mod_alias.so

WSGIPassAuthorization On
       <Directory /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy>
               <Files "wsgi.py">
                       Order deny,allow
                       Allow from all
               </Files>
           Options Indexes FollowSymLinks
           AllowOverride All
           Require all granted
       </Directory>
WSGIPassAuthorization On

WSGIScriptAlias / /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy/wsgi.py
       <IfModule mod_dir.c>
           DirectoryIndex index.php index.pl index.cgi index.html index.xhtml index.htm
       </IfModule>
</VirtualHost>
```

The path /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy/wsgi.py is obtained from:
```
root@saltapi:~# find / -name "wsgi.py"
/usr/local/lib/python2.7/dist-packages/cheroot/wsgi.py
/usr/lib/python3/dist-packages/twisted/web/wsgi.py
/usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy/wsgi.py
/usr/lib/python2.7/dist-packages/tornado/wsgi.py
```

For Apache path to /salt/netapi/rest_cherrypy has to be picked.

Restart Apache:
```
systemctl restart apache2
```

8) Curl to generate tokens to raise a API call further

```
curl -isSk https://localhost:8000/login     -H 'Accept: application/x-yaml'     -d username=saltapi     -d password=saltapi     -d eauth=pam

HTTP/1.1 200 OK
Content-Length: 158
Access-Control-Expose-Headers: GET, POST
Vary: Accept-Encoding
Server: CherryPy/17.4.2
Allow: GET, HEAD, POST
Access-Control-Allow-Credentials: true
Date: Fri, 13 Sep 2019 15:54:41 GMT
Access-Control-Allow-Origin: *
X-Auth-Token: fb08acedbc3ab638896d8005d4dddba080bf66d4
Content-Type: application/x-yaml
Set-Cookie: session_id=fb08acedbc3ab638896d8005d4dddba080bf66d4; expires=Sat, 14 Sep 20                                                                                          19 01:54:41 GMT; Max-Age=36000; Path=/

return:
- eauth: pam
  expire: 1568433281.376168
  perms:
  - .*
  start: 1568390081.376167
  token: fb08acedbc3ab638896d8005d4dddba080bf66d4
  user: saltapi

```

Error faced when curled via Apache (because of the SSL issued mentioned earlier):
```
root@saltapi:~# curl -isSk https://localhost:80/login     -H 'Accept: application/x-yaml'     -d username=saltapi     -d password=saltapi     -d eauth=pam
curl: (35) error:1408F10B:SSL routines:ssl3_get_record:wrong version number
```

But for both(direct salt-api; apache) HTTP curls works fine: 
```
root@saltapi:~# curl -isSk http://localhost:80/login     -H 'Accept: application/x-yaml'     -d username=saltapi     -d password=saltapi     -d eauth=pam
HTTP/1.1 200 OK
Date: Fri, 13 Sep 2019 16:00:36 GMT
Server: Apache/2.4.29 (Ubuntu)
Content-Length: 158
Access-Control-Expose-Headers: GET, POST
Vary: Accept-Encoding
Allow: GET, HEAD, POST
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
X-Auth-Token: d4872db106b239f48f077ac4ca78786d26ae65da
Set-Cookie: session_id=d4872db106b239f48f077ac4ca78786d26ae65da; expires=Sat, 14 Sep 2019 02:00:37 GMT; Max-Age=36000; Path=/
Content-Type: application/x-yaml

return:
- eauth: pam
  expire: 1568433637.060853
  perms:
  - .*
  start: 1568390437.060852
  token: d4872db106b239f48f077ac4ca78786d26ae65da
  user: saltapi
```

For test pings, salt minion and master have to be set up(if not already):
```
curl -L http://bootstrap.saltstack.com -o install-salt.sh
sh install-salt.sh -M -A 127.0.0.1
```
The above might not be necessary, but just in case if the minion isn't detected by the master.
```
root@saltapi:~# salt-call --local test.ping
local:
    True
```
Now the minion- saltapi is installed and needs to be accepted by the master (which is in the same machine)
```
root@saltapi:~# salt-key
Accepted Keys:
Denied Keys:
Unaccepted Keys:
saltapi
Rejected Keys:
root@saltapi:~# salt-key -f saltapi
Unaccepted Keys:
saltapi:  68:29:57:c5:eb:31:08:24:23:95:2c:48:10:61:22:3b:ba:94:d9:c9:42:1a:1a:fb:d0:bb:50:8b:81:51:da:a4
root@saltapi:~# salt-call --local key.finger
local:
    68:29:57:c5:eb:31:08:24:23:95:2c:48:10:61:22:3b:ba:94:d9:c9:42:1a:1a:fb:d0:bb:50:8b:81:51:da:a4
root@saltapi:~# salt-key -a saltapi
The following keys are going to be accepted:
Unaccepted Keys:
saltapi
Proceed? [n/Y] Y
Key for minion saltapi accepted.
root@saltapi:~# salt-key
Accepted Keys:
saltapi
Denied Keys:
Unaccepted Keys:
Rejected Keys:
root@saltapi:~#
```

Copy the token value from the output and include it in subsequent requests:
```
root@saltapi:~# curl -isSk http://localhost:80     -H 'Accept: application/x-yaml'     -H 'X-Auth-Token: d4872db106b239f48f077ac4ca78786d26ae65da'    -d client=local     -d tgt='*'     -d fun=test.ping
HTTP/1.1 200 OK
Date: Fri, 13 Sep 2019 16:42:42 GMT
Server: Apache/2.4.29 (Ubuntu)
Content-Length: 24
Access-Control-Expose-Headers: GET, POST
Cache-Control: private
Vary: Accept-Encoding
Allow: GET, HEAD, POST
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Set-Cookie: session_id=d4872db106b239f48f077ac4ca78786d26ae65da; expires=Sat, 14 Sep 2019 02:42:42 GMT; Max-Age=36000; Path=/
Content-Type: application/x-yaml

return:
- saltapi: true
```

The configured minion just returns True response for the ping test via Apache server. 

<br />
#####################################################################################################
<br />


Now with standalone [rest_wsgi](https://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_wsgi.html), there's a issue that is still open and yet to be resolved: https://github.com/saltstack/salt/issues/40801
This issue is due to threads hogging the memory. 
So the workaround for that issue would be adding the following lines to Apache config:

```
WSGIDaemonProcess localhost processes=1 threads=1 display-name=%{GROUP}
WSGIProcessGroup localhost
```
Change your threads accordingly.

Now the config files look like: 

Apache
```
root@saltapi:~# cat /etc/apache2/sites-enabled/000-default.conf
<VirtualHost *:80>
       ServerName localhost
       ServerAlias localhost
       ServerAdmin webmaster@localhost
       DocumentRoot /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy
       ErrorLog /var/log/apache2/error.log
       CustomLog /var/log/apache2/access.log combined

LogLevel debug
LoadModule alias_module /usr/lib/apache2/modules/mod_alias.so

WSGIPassAuthorization On
       <Directory /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy>
               <Files "wsgi.py">
                       Order deny,allow
                       Allow from all
               </Files>
           Options Indexes FollowSymLinks
           AllowOverride All
           Require all granted
       </Directory>
WSGIPassAuthorization On

WSGIDaemonProcess localhost processes=1 threads=1 display-name=%{GROUP}
WSGIProcessGroup localhost

WSGIScriptAlias / /usr/lib/python2.7/dist-packages/salt/netapi/rest_cherrypy/wsgi.py
       <IfModule mod_dir.c>
           DirectoryIndex index.php index.pl index.cgi index.html index.xhtml index.htm
       </IfModule>
</VirtualHost>

```

Salt master (ports are user defined):
```
root@saltapi:~# cat /etc/salt/master.d/salt-api.conf
rest_wsgi:
  port: 8001

external_auth:
  pam:
    saltapi:
      - .*

```

Dont forget to restart apache2, salt-master, and salt-api.

Curl to generate tokens:
```
root@saltapi:~# curl -isSk http://localhost:80/login     -H 'Accept: application/x-yaml'     -d username=saltapi     -d password=saltapi     -d eauth=pam
HTTP/1.1 200 OK
Date: Fri, 13 Sep 2019 18:18:49 GMT
Server: Apache/2.4.29 (Ubuntu)
Content-Length: 158
Access-Control-Expose-Headers: GET, POST
Vary: Accept-Encoding
Allow: GET, HEAD, POST
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
X-Auth-Token: a6e5c19613fb55b7d2293fef04c628bb3d4f96bd
Set-Cookie: session_id=a6e5c19613fb55b7d2293fef04c628bb3d4f96bd; expires=Sat, 14 Sep 2019 04:18:49 GMT; Max-Age=36000; Path=/
Content-Type: application/x-yaml

return:
- eauth: pam
  expire: 1568441929.720754
  perms:
  - .*
  start: 1568398729.720754
  token: a6e5c19613fb55b7d2293fef04c628bb3d4f96bd
  user: saltapi

```


Use the token to ping: 
```
root@saltapi:~# curl -isSk http://localhost:80     -H 'Accept: application/x-yaml'     -H 'X-Auth-Token: a6e5c19613fb55b7d2293fef04c628bb3d4f96bd'    -d client=local     -d tgt='*'     -d fun=test.ping
HTTP/1.1 200 OK
Date: Fri, 13 Sep 2019 18:20:02 GMT
Server: Apache/2.4.29 (Ubuntu)
Content-Length: 24
Access-Control-Expose-Headers: GET, POST
Cache-Control: private
Vary: Accept-Encoding
Allow: GET, HEAD, POST
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Set-Cookie: session_id=a6e5c19613fb55b7d2293fef04c628bb3d4f96bd; expires=Sat, 14 Sep 2019 04:20:02 GMT; Max-Age=36000; Path=/
Content-Type: application/x-yaml

return:
- saltapi: true

```




References: 

https://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_cherrypy.html

https://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_wsgi.html

https://bitbucket.org/cherrypy/cherrypy/downloads/

https://github.com/saltstack/salt/issues/40801

https://medium.com/@shubham_arora/vertical-scaling-a-python-application-running-on-mod-wsgi-8894ecb5714f

https://modwsgi.readthedocs.io/en/develop/user-guides/quick-configuration-guide.html
