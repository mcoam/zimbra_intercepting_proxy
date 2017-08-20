# ZmProxy

This software is used to intercept and apply modifications to the traffic between a Zimbra Proxy and Zimbra Mailboxes. If you don't know what a Zimbra Proxy is, You can read about it here: [https://wiki.zimbra.com/wiki/Zimbra_Proxy_Guide](https://wiki.zimbra.com/wiki/Zimbra_Proxy_Guide)

This work for all kind of client access:

* POP3
* IMAP
* Webmail
* ActiveSync
* Zimbra Outlook Connector

## What this try to solve?

### Zimbra Migrations
Suppose you need to move a lot of users and data from one Zimbra Platform to another, like we do at [ZBox](http://www.zboxapp.com), and given the size of the migration, you can't move all the mailboxes at once, so you have to do it in groups.

This procedure have the following inconvenients:

* You have to update the configuration of the clients for all the migrated users,
* Your users will need to learn a new Webmail URL,
* It's not transparent for the end user,
* It's a lot of work for you

### Network and OpenSource Deployments
Not a hot topic for Zimbra Inc., sorry guys, but lets be honest about it, some companies can't afford Zimbra Network for all the employees, so they use two setup platform.

The main problem with this is that you have to configure your clients with to kind of information.

## Tutorial
This tutorial is based on the following setup:

* One Zimbra Mailbox v6, that we want to migrate to,
* Multi Server Zimbra, with 1 Zimbra Proxy, and 2 Zimbra Mailboxes

Also, the new Zimbra Proxy should be running on CentOS 7.

### 1. Install Docker on the Zimbra Proxy Server

```
$ yum install epel-release -y
$ yum install docker -y
$ systemctl enable docker
$ systemctl start docker
```

### 2. Configure Zimbra Nginx Templates
This config is going to use a running Docker listen on the `9090` port.

 ```
 # Backup directory
 $ cp -a /opt/zimbra/conf/nginx/templates /opt/zimbra/conf/nginx/templates.backup
 $ curl -k https://raw.githubusercontent.com/ITLinuxCL/zimbra_intercepting_proxy/master/examples/nginx.conf.mail.template > /opt/zimbra/conf/nginx/templates/nginx.conf.mail.template
 $ curl -k https://raw.githubusercontent.com/ITLinuxCL/zimbra_intercepting_proxy/master/examples/nginx.conf.web.template > /opt/zimbra/conf/nginx/templates/nginx.conf.web.template
 $ curl -k https://raw.githubusercontent.com/ITLinuxCL/zimbra_intercepting_proxy/master/examples/nginx.conf.zmlookup.template > /opt/zimbra/conf/nginx/templates/nginx.conf.zmlookup.template
 $ chown zimbra.zimbra -R /opt/zimbra/conf/nginx/templates/
 ```

### 3. Create and Admin User
We need an admin user to lookup the mailbox for the account, this admin needs to have a **non-expiring token**.

```
$ zmprov ca zmproxy@example.com Password \
  zimbraIsAdminAccount TRUE \
  zimbraAdminAuthTokenLifetime 100d \
  zimbraAuthTokenLifetime 100d
```
### 4. Prerequisites

* On mailbox backends: check the service ports on listen the service (pop, imap, web)

Ej: Standard ports: 110, 143, 80

```
[root@zimbra-mbx2 ~]# netstat -tlpn |egrep "143|110|80"
tcp        0      0 0.0.0.0:110             0.0.0.0:*               LISTEN      2166/java
tcp        0      0 0.0.0.0:143             0.0.0.0:*               LISTEN      2166/java
tcp        0      0 0.0.0.0:80            0.0.0.0:*               LISTEN      2166/java
```

or Mailbox (if proxy configured): 7110, 7143, 8080

```
[root@zimbra-mbx2 ~]# netstat -tlpn |egrep "143|110|80"
tcp        0      0 0.0.0.0:7110             0.0.0.0:*               LISTEN      2166/java
tcp        0      0 0.0.0.0:7143             0.0.0.0:*               LISTEN      2166/java
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      2166/java
```

* On default mailbox configure `zimbraMailTransport` for user map.

```
## Map to Z8 - mailbox2
zmprov ga raguirre@example.com |grep Transport
zimbraMailTransport: lmtp:zimbra-mbx2.example.com:7025

## Map to Z8 - mailbox1
zmprov ga cmora@example.com |grep Transport
zimbraMailTransport: lmtp:zimbra-mbx1.example.com:7025

## Map to Zimbra 6 - single server
zmprov ga plopez@example.com |grep Transport
zimbraMailTransport: lmtp:mail.example.com:7025

```

### 5. Start the Container

```
# Interactive Mode

$ docker run -d --name zimbra_zip --dns=10.13.28.16 -p 9090:9090 \
-e ZIMBRA_USER=zimbra@example.com \
-e ZIMBRA_PASSWORD=Password \
-e NAMESERVERS=10.13.28.16 \
-e ZIMBRA_SOAP=https://mail.example.com:7071/service/admin/soap \
-e DEFAULT_MAILBOX_IP=10.13.210.43 \
-e MAILBOXES_MAPPING='zimbra-mbx2.example.com:10.13.138.81:8080:110:143:true;zimbra-mbx1.example.com:10.13.138.80:8080:110:143:true;mail.example.com:10.13.210.43:80:110:143' \
-e PREFIX_PATH=/zimbra \
-e VERBOSE=true itlinuxcl/zimbra_zip:0.6.16

# Background just change this line
$ docker run -d --name zimbra_zip --dns=192.168.80.81 -p 9090:9090 \
  ......
```

You can check de running docker in background with:

```
docker logs zimbra_zip -f
```

About the variables:

* `ZIMBRA_USER`, is the admin user,
* `ZIMBRA_PASSWORD`, password for the admin user,
* `NAMESERVERS`, an IP address of a DNS server that can resolv all mailboxes, including old and new,
* `ZIMBRA_SOAP`, url of a mailbox where to do the lookups,
* `DEFAULT_MAILBOX_IP`, one of the mailboxes on the new platform. The formats is:
* `PREFIX_PATH`, the Zimbra 6 use a prefix when sending requests,
* `MAILBOXES_MAPPING`, this holds the information of the port used in every mailbox.

The syntax of `MAILBOXES_MAPPING` is:

```
FQDN,SERVER_IP,WEB_PORT,POP_PORT,IMAP_PORT:true:FQDN,SERVER_IP,WEB_PORT,POP_PORT,IMAP_PORT:REMOVE_PREFIX;
```

Where:

* `FQDN`, hostname `(zmhostname command)`
* `SERVER_IP`, ip of a mailbox
* `WEB_PORT,POP_PORT,IMAP_PORT`, ports where listen the webmail, pop3, and imap services (Check Prerequisites)
* `REMOVE_PREFIX`, it ZmProxy shoud remove the `/zimbra` prefix for this mailbox

You **must** list all the mailboxes here.

### 6. Re-configure Zimbra Proxy and restart

```
$ /opt/zimbra/libexec/zmproxyconfgen
$ /opt/zimbra/bin/zmproxyctl restart
```

### 7. Check status
If you run the container in Interactive mode, you should see the logs on your screen, if not
you can run the following command:

```
$ docker logs zimbra_zip -f
```

### 8. Useful commands

* `docker ps`, List docker status (container id dcffe71be046)
* `docker rm dcffe71be046`, Remove docker container
* `docker start zimbra_zip`, Start docker container
* `docker stop zimbra_zip`, Stop docker container
* `docker logs -f dcffe71be046 |grep user@domain.com`, Check container log for specific user

### 9. Configure systemd service for docker image

To enable or start the service, systemd needs to be reloaded our new service files:

* Create file `/etc/systemd/system/docker-zproxy.service` and (copy/paste) the next setence:

```
[Unit]
Description=ZBox zimbra_zip
After=docker.service
Requires=docker.service

[Service]
Restart=always
#ExecStartPre=-/usr/bin/docker stop zimbra_zip
ExecStartPre=-/usr/bin/docker rm zimbra_zip
ExecStart=/usr/bin/docker run --name zimbra_zip --dns=10.13.28.16 -p 9090:9090 -e ZIMBRA_USER=zimbra@example.com -e ZIMBRA_PASSWORD=Passowrd -e NAMESERVERS=10.13.28.16 -e ZIMBRA_SOAP=https://mail.example.com:7071/service/admin/soap -e DEFAULT_MAILBOX_IP=10.13.210.43 -e MAILBOXES_MAPPING='zimbra-mbx2.example.com:10.13.138.81:8080:110:143:true;zimbra-mbx1.example.com:10.13.138.80:8080:110:143:true;mail.example.com:10.13.210.43:80:110:143' -e PREFIX_PATH=/zimbra -e VERBOSE=true itlinuxcl/zimbra_zip:0.6.16
ExecStop=/usr/bin/docker stop zimbra_zip

[Install]
WantedBy=multi-user.target
```

And execute

```
systemctl daemon-reload
systemctl start docker-zproxy
```

Finally, to confirm that everything worked:

```
$ systemctl status docker-zproxy
$ docker ps
```

* For `stop` service:  `systemctl stop docker-zproxy`
* For `start` service:  `systemctl start docker-zproxy`
* For `status` service:  `systemctl status docker-zproxy`

For remove or disable the service

* Rename the file `mv /etc/systemd/system/docker-zproxy.service /etc/systemd/system/docker-zproxy.service.old`
* Run: `systemctl reset-failed`

### 10. Monitoring

For monitoring we have `monit` service. The next config check port `9090` and start service if port is down.

* Install service

```
yum install monit -y
```

* Create file `/etc/monit.d/docker-zproxy` and (copy/paste) the next setence

```
check host localhost with address localhost
    start program = "/usr/sbin/service docker-zproxy start"
    stop program  = "/usr/sbin/service docker-zproxy stop"
    alert mcoa@example.com
    if failed port 9090
      with timeout 3 seconds
      then restart
```

* Check and reload config

```
monit -t
service monit restart
```


## Thanks

* To the Zimbra folks for a great product, and
* [@igrigorik](http://twitter.com/igrigorik) for [em-proxy](https://github.com/igrigorik/em-proxy)

## Contributing

1. Fork it ( https://github.com/pbruna/zimbra_intercepting_proxy/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
