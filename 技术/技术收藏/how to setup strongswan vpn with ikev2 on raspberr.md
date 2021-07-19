how to setup strongswan vpn with ikev2 on raspberry pi

[strongswan](https://strongswan.org/) is an opensource, ipsec-based vpn server, available for almost all operating systems, and it runs smoothly on raspberry pi. if you have set up [pihole](https://pi-hole.net/) on your pi, you can block unwanted advertisement while you are away from home. or, you just want to access your local network from outside. whatever your goal is, here's how to install and configure strongswan with secure ikev2 support on your raspberry pi.

### table of contents

- [setup](#setup)
- [configuration](#configuration)
- [port forwarding](#portforwarding)
- [iptables](#iptables)
- [.mobileconfig for ios and macos](#mobileconfigforiosandmacos)

log in to your raspberry using ssh, and run this command to install strongswan:  
`$ sudo apt-get install strongswan libcharon-extra-plugins`

* * *

## /etc/strongswan.conf

now, we want to edit our configuration files. first, we begin with `/etc/strongswan.conf`, open the file in nano `$ sudo nano /etc/strongswan.conf` and fill in:

```
charon {
    load_modular=yes
    dns1=74.82.42.42
    dns2=2001:470:20::2
    plugins {
        include strongswan.d/charon/*.conf
    }
}

include strongswan.d/*.conf

```

please update `dns1` and `dns2` with the ipv4 and ipv6 addresses of your local dns server / pihole, or any public dns server. save and exit nano (`[ctrl]-[o]`, and `[ctrl]-[x]`)

## /etc/ipsec.conf

next, open `/etc/ipsec.conf` in nano: `$ sudo nano /etc/ipsec.conf` and fill it with:

```
config setup
    strictcrlpolicy=no
    uniqueids=no

conn %default
    mobike=yes
    dpdaction=clear
    dpddelay=35s
    dpdtimeout=200s
    fragmentation=yes

conn iOS-IKEV2
    auto=add
    keyexchange=ikev2
    eap_identity=%any
    left=%any
    leftsubnet=0.0.0.0/0
    rightsubnet=192.168.1.0/24
    leftauth=psk
    leftid=%any
    right=%any
    rightsourceip=192.168.1.0/24
    rightauth=eap-mschapv2
    rightdns=74.82.42.42,2001:470:20::2
    rightid=%any

```

lines, you probably want to change, to adapt your network configuration:

- `rightsubnet`: this subnet will be used for connected clients
- `rightsourceip`: these ip addresses will be used for connected clients
- `rightdns`: your desired servers for dns resolving

## /etc/ipsec.secrets

final configuration file is `/etc/ipsec.secrets`; this contains your credentials to connect to your strongswan vpn server. run `sudo nano /etc/ipsec.secrets`. based on this sample configuration, you should enter your usernames and passwords for your clients:

```
include /var/lib/strongswan/ipsec.secrets.inc

: PSK "strongSwanVPN"
iphone : EAP "c$y[}:M?#GEf}VqiSP4Ukvg=]eIKRt\,CUE><=uilonDLh/{N}-'F9R@wXd3=8Xn"
mac : EAP "sbTH!)98)4$fPKtAZJF$x\/49ukUW:2s*zZfDHL*;g6>NKEWCh\td=t&j^hP0p7t"`

```

enter as many user/password combination, as you want to. optional: change the pre-shared key (PSK) to any value. if you quickly want to get a secure password, run this command: `openssl rand -base64 48`

* * *

final configuration is done in `/etc/sysctl.conf`. open it in nano (`$ sudo nano /etc/sysctl.conf`) and change two lines to match:

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

```

after editing, run `$ sudo sysctl -p` to reload the configuration.

* * *

please setup port forwarding for both ipv4 and ipv6 in your router's settings, using this example:

- **isakmp**: forward port 500 udp to your raspberry pi (i.e. 192.168.0.2)
- **ipsec nat traversal**: forward port 4500 udp to our raspberry pi
- **encapsulating security payload**: forward esp to your raspberry pi

* * *

on your raspberry pi, you should create a script, that will update it's iptables after each reboot:

```
$ mkdir ~/etc; nano ~/etc/strongswan_iptables.sh
$ chmod +x ~/etc/strongswan_iptables.sh

```

fill it with:

```
#!/bin/sh
sudo iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 500 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4500 -j ACCEPT

```

again, change the network address, if necessary. it's the same value from above (`rightsubnet` and `rightsourceip`).

in your `crontab` you should add this line (run `$ crontab -e` to edit):

```
@reboot /home/pi/etc/strongswan_iptables.sh

```

now, you are done with setting up strongswan and you should reboot your pi: `sudo reboot`

* * *

feel free to use this example `vpn.mobileconfig` to setup vpn on your (apple) client devices. please remember, to edit:

- `AuthName`
- `AuthPassword`
- `RemoteAddress`
- `RemoteIdentifier`
- `SharedSecret`

```
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <!-- Home VPN -->
        <dict>
            <key>IKEv2</key>
            <dict>
                <key>AuthName</key>
                <string>iphone</string>        <!-- this is the username from ipsec.secrets -->
                <key>AuthPassword</key>
                <string>c$y[}:M?#GEf}VqiSP4Ukvg=]eIKRt\,CUE><=uilonDLh/{N}-'F9R@wXd3=8Xn</string>        <!-- this is the password from ipsec.secrets -->
                <key>AuthenticationMethod</key>
                <string>SharedSecret</string>
                <key>ChildSecurityAssociationParameters</key>
                <dict>
                    <key>DiffieHellmanGroup</key>
                    <integer>2</integer>
                    <key>EncryptionAlgorithm</key>
                    <string>3DES</string>
                    <key>IntegrityAlgorithm</key>
                    <string>SHA1-96</string>
                    <key>LifeTimeInMinutes</key>
                    <integer>1440</integer>
                </dict>
                <key>DeadPeerDetectionRate</key>
                <string>Medium</string>
                <key>DisableMOBIKE</key>
                <integer>0</integer>
                <key>DisableRedirect</key>
                <integer>0</integer>
                <key>EnableCertificateRevocationCheck</key>
                <integer>0</integer>
                <key>EnablePFS</key>
                <integer>0</integer>
                <key>ExtendedAuthEnabled</key>
                <true/>
                <key>IKESecurityAssociationParameters</key>
                <dict>
                    <key>DiffieHellmanGroup</key>
                    <integer>2</integer>
                    <key>EncryptionAlgorithm</key>
                    <string>3DES</string>
                    <key>IntegrityAlgorithm</key>
                    <string>SHA1-96</string>
                    <key>LifeTimeInMinutes</key>
                    <integer>1440</integer>
                </dict>
                <key>LocalIdentifier</key>
                <string>one.nerd.vpn.iphone</string>
                <key>RemoteAddress</key>
                <string>1.2.3.4</string>        <!-- enter your home ip address here -->
                <key>RemoteIdentifier</key>
                <string>example.com</string>        <!-- enter your domain name here, i.e. your dyndns address -->
                <key>SharedSecret</key>
                <string>strongSwanVPN</string>        <!-- this is the pre-shared key from ipsec.secrets -->
                <key>UseConfigurationAttributeInternalIPSubnet</key>
                <integer>0</integer>
                <key>OnDemandEnabled</key>
                <integer>0</integer>
            </dict>
            <key>IPv4</key>
            <dict>
                <key>OverridePrimary</key>
                <integer>1</integer>
            </dict>
            <key>PayloadDescription</key>
            <string>Configures VPN settings for iphone</string>
            <key>PayloadDisplayName</key>
            <string>TutorialVPN</string>
            <key>PayloadIdentifier</key>
            <string>com.apple.vpn.managed.6394414C-E8F6-4063-9CC7-80099D37E47D</string>
            <key>PayloadType</key>
            <string>com.apple.vpn.managed</string>
            <key>PayloadUUID</key>
            <string>6394414C-E8F6-4063-9CC7-80099D37E47D</string>
            <key>PayloadVersion</key>
            <real>1</real>
            <key>Proxies</key>
            <dict>
                <key>HTTPEnable</key>
                <integer>0</integer>
                <key>HTTPSEnable</key>
                <integer>0</integer>
                <key>ProxyAutoConfigEnable</key>
                <integer>0</integer>
                <key>ProxyAutoDiscoveryEnable</key>
                <integer>0</integer>
            </dict>
            <key>UserDefinedName</key>
            <string>Home VPN</string>
            <key>VPNType</key>
            <string>IKEv2</string>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>VPN</string>
    <key>PayloadIdentifier</key>
    <string>one.nerd.iphone.vpn.7E1B78A5-D482-4CA6-9BFE-28299456A795</string>
    <key>PayloadOrganization</key>
    <string>nerd.one</string>
    <key>PayloadRemovalDisallowed</key>
    <true/>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>F1049BB3-8E20-48EE-A134-F30A5FA17E83</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>

```

if you want to enhance you vpn configuration file with ondemand rules, please continue reading here: [vpn on demand configuration profiles for ios and macos explained](https://nerd.one/vpn-on-demand-configuration-profiles-for-ios-and-macos-explained/)