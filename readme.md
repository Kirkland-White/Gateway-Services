###### Gateway Documentation
This document will contain instructions on how to create a network appliance with an Ubuntu Server.
Our network topology consists of a client, the network appliance we built running Ubuntu Server with 2 NICs (one for the internal network and one for external),
and a connection to our ISP. The network appliance provides DNS services, DHCP services, Firewall Services and Proxy services.
This Document will not cover OS install. The inward facing NIC on our server has an IP of 10.0.0.250 and the outward on is 192.168.0.12


The format of this document is as follows:

- DNS Server Setup
- DHCP Server Setup
- Firewall Setup
- Proxy Server Setup

#### DNS Server Setup

To start, install the DNS software bind9 to your server by typing this command


sudo apt install bind9


Then, install dnsutils.


sudo apt install dnsutils


The configuration files will be stored in the /etc/bind directory.

To create a new zone and make our server a primary server, we edit the named.conf.local file and enter this:


zone "sysninja" {
    type master;
    file "/etc/bind/db.sysninja";
};


After that, make a copy of the existing zone file (db.local) to create a new file. Give it the name db.sysninja. We will use the other zone file as a template.
Using our new file, replace localhost with sysninja. and edit it so it looks like this:


;
; BIND data file for sysninja.
;
$TTL	604800
@	IN	SOA	sysninja. root.sysninja. (
			      2		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	ns.sysninja.
@	IN	A	192.168.0.12
@	IN	AAAA	::1
ns	IN	A	192.168.0.12
*	IN	A	99.99.99.99
test	IN	CNAME	sysninja.


After that, restart the BIND9 service using the command:


sudo systemctl restart bind9.service



#### DHCP Server Setup

To start, install isc-dhcp-server using this command:


sudo apt-get install isc-dhcp-server


You will get an error message after the installation but that is fine. Now we want to edit the dhcpd.conf file in /etc/dhcp and add this below the header comments:


option domain-name ".sysninja";
option domain-name-servers 10.0.0.250;
option broadcast-address 10.0.0.255;
default-lease-time 600;
max-lease-time 7200;

subnet 10.0.0.0 netmask 255.255.255.0 {
range 10.0.0.1 10.0.0.100;
range 10.0.0.201 10.0.0.215;
}


We use the network of the client facing NIC for the configurations. We also put a range of IPs we want the DHCP server to hand out.
After that, restart the dhcp server by using this command:


sudo service isc-dhcp-server restart


Your DHCP server should be active.

#### Firewall Setup

Ubuntu Server comes with the default Firewall configuration tool called ufw. ufw is designed to make firewall rules easier to implement into ip tables.

To enable ufw, simply use the command:


sudo ufw enable


You can use ufw to configure ports for specific IPs and allow, deny and reject packets.
For example, this command would allow ssh from 192.168.0.2 from any address on port 22:


sudo ufw allow proto tcp from 192.168.0.2 to any port 22


We can also do IP Masquerading. To do this, we have to edit the file ufw in /etc/default to allow packet forwarding. Change the DEFAULT_FORWARD_POLICY to ACCEPT.


DEFAULT_FORWARD_POLICY="ACCEPT"


We then edit /etc/ufw/sysctl.conf and uncomment	the line:


net/ipv4/ip_forward=1


After that, we add this to the before.rules file in /etc/ufw just under the header comments:


# nat Table rules
*nat
:POSTROUTING ACCEPT [0:0]

#Forward traffic from ens39 through ens33
-A POSTROUTING -s 10.0.0.0/24 -o ens33 -j MASQUERADE

#don't delete the 'COMMIT' line or these nat Table rules won't be processed
COMMIT


The subnet will be of your Internal network and ens33 will be replaced by the name of you NIC facing your ISP

Now use this command to disable and re-enable ufw:


sudo ufw disable && sudo ufw enable


You should now have IP masquerading enabled.


#### Proxy Server Setup


To start off, install Squid using the command:


sudo apt install squid


Squid is configured by editing the squid.conf file. This file has a lot of lines in it and it is suggested to make a back up of it just to have for reference.

Add this line to the acl section of the squid.conf file:


acl localnet src 10.0.0.0/24


Replace the IP address with the IP subnet of your private network.


We can configure squid as a transparent proxy by adding this iptables rule:


sudo iptables -t nat -I PREROUTING -p tcp -s 10.0.0.0/24 --dport 80 -j REDIRECT --to-port 3128


The default port for squid is 3128, so if you change which port squid operates on, the iptables rule will have to use your custom port.
now if you visit a http website on your client machine, it will go through the proxy. 

Now we are going to configure Squid so that it will redirect from the website http:www.neverssl.com to google.com

In the acl section of squid.conf, type this:


acl localnet badsites dstdomain .neverssl.com


In the deny_info section, add this:


deny_info http://www.google.com localnet


In the http_reply_access section, add:


http_reply_access deny badsites localnet


Then, finally restart the server with:


service squid stop && service squid start


Now, the proxy server will redirect traffic trying to get to www.neverssl.com to google.com