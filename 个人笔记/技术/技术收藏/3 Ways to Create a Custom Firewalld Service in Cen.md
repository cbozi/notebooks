3 Ways to Create a Custom Firewalld Service in CentOS 7.6

![3 Ways to Create a Custom Firewalld Service in CentOS 7.6](https://1.bp.blogspot.com/-MI5G8l5-xTM/XcfKoj4rD0I/AAAAAAAAG4s/1aRcWbojSYYlLkvtv0sb2YYanh--SQY1wCLcBGAsYHQ/s1600/3-ways-to-create-a-firewalld-service.jpg "3 Ways to Create a Custom Firewalld Service in CentOS 7.6")

[Firewalld](https://firewalld.org/) is a firewall management tool for Linux operating systems licensed under GNU General Public License 2. Firewalld is the default firewall management tool in RHEL 7 onwards, where it replaces the legacy firewall management tool iptables. Firewalld is a dynamically managed firewall with support for network zones, IPv4, IPv6, ethernet bridges and IP sets.

In this article, we will explore the 3 ways to create a custom firewalld service in CentOS 7.6.

## This Article Provides:

- [System Specification](#point1)
- [1) Create a Custom Firewalld Service using CLI](#point2)
- [2) Create a Custom Firewalld Service from XML file](#point3)
- [3) Create a Custom Firewalld Service from Definition File](#point4)

## System Specification:

Consider a scenario where we are running an Oracle Database 12c instance on CentOS 7.6 server.

Default Oracle Listener uses the service port 1521/tcp. We have also configured another Oracle Listener service that is using port 1522/tcp.

In short, we have two Oracle listeners running on ports 1521/tcp and 1522/tcp simultaneously.

Our objective is to create a custom firewalld service to control access from network to our Oracle Listener ports.

## 1) Create a Custom Firewalld Service using CLI:

In this method, we will create a custom firewalld service using firewall-cmd command.

Create a new service for Oracle Listener ports.

```
[root@dev-03 ~]# firewall-cmd --permanent --new-service=oranet
success
```

Add long description of the service.

```
[root@dev-03 ~]# firewall-cmd --permanent --service=oranet \
> --set-description="Oracle Listener Service"
success
```

Add short description of the service.

```
[root@dev-03 ~]# firewall-cmd --permanent --service=oranet \
> --set-short=oranet
success
```

Add Oracle Listener service ports.

```
[root@dev-03 ~]# firewall-cmd --permanent --service=oranet --add-port=1521/tcp
success
[root@dev-03 ~]# firewall-cmd --permanent --service=oranet --add-port=1522/tcp
success
```

Reload firewalld configurations.

```
[root@dev-03 ~]# firewall-cmd --reload
success
```

Display configurations of our firewalld service.

```
[root@dev-03 ~]# firewall-cmd --info-service=oranet
oranet
  ports: 1521/tcp 1522/tcp
  protocols:
  source-ports:
  modules:
  destination:
```

We can add more settings to our service in similar way. You can refer to [Firewalld Documentation](https://firewalld.org/documentation/howto/add-a-service.html) for more details.

## 2) Create a Custom Firewalld Service from XML file:

In this method, we will define the firewalld service settings in an XML file and then use firewall-cmd command to create a custom firewalld service.

```
[root@dev-03 ~]# vi ~/oranet.xml
```

and add following XML code therein.

```
<?xml version="1.0" encoding="utf-8"?>
<service>
 <short>oranet</short>
 <description>Oracle Listener Service</description>
 <port protocol="tcp" port="1521" />
 <port protocol="tcp" port="1522" />
</service>
```

Now use firewall-cmd command to create firewalld service.

```
[root@dev-03 ~]# firewall-cmd --permanent --new-service-from-file=oranet.xml
success
```

Reload firewalld configurations and check oranet service.

```
[root@dev-03 ~]# firewall-cmd --reload
success
[root@dev-03 ~]# firewall-cmd --info-service=oranet
oranet
  ports: 1521/tcp 1522/tcp
  protocols:
  source-ports:
  modules:
  destination:
```

## 3) Create a Custom Firewalld Service from Definition File:

This method is normally used by packages during installation to create their respective firewalld services.

In this method, we create an firewalld service definition file in firewalld configuration directory.

```
[root@dev-03 ~]# vi /etc/firewalld/services/oranet.xml
```

Add following XML code therein.

```
<?xml version="1.0" encoding="utf-8"?>
<service>
 <short>oranet</short>
 <description>Oracle Listener Service</description>
 <port protocol="tcp" port="1521" />
 <port protocol="tcp" port="1522" />
</service>
```

Reload firewalld configurations and check service oranet service.

```
[root@dev-03 ~]# firewall-cmd --reload
success
[root@dev-03 ~]# firewall-cmd --info-service=oranet
oranet
  ports: 1521/tcp 1522/tcp
  protocols:
  source-ports:
  modules:
  destination:
```

We have explored all 3 ways to create a custom firewalld service in centos 7.