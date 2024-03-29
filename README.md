
# VPN


## NAT
Basic NAT config with FreeBSD

## Test computer (Border NAT)
```
$ dmesg | less: 
FreeBSD 8.0-STABLE-201004 #0: Mon Apr 5 15:59:06 UTC 2010 
CPU: Intel(R) Pentium(R) 4 CPU 3.20GHz (3200.01-MHz K8-class CPU) 
real memory = 536870912 (512 MB) 
age0: mem 0xfeac0000-0xfeafffff irq 17 at device 0.0 on pci2 
rl0: port 0xe800-0xe8ff mem 0xfebffc00-0xfebffcff irq 19 at device 0.0 on pci4 
```
## Router 
Up to date with the latest security advisories
```
# freebsd-update fetch
# freebsd-update install
```
Update packages
```
# pkg upgrade
```
Install dependencies
```
# pkg install iperf netstat
# pkg install openvpn
```

## Scheme
![scheme](images/scheme.jpg)


IP Pool for nat is `192.168.133.0/24`

DNS Server: `192.168.133.1`

LAN network `192.168.0.0/16`

LAN Range `192.168.150.50` -> `192.168.200.254`

`xn0` - external interface (WAN)

`xn1` - internal interface (LAN)


## Config NAT

### FreeBSD Network Configuration

Open /etc/rc.conf in your favorite editor. You need to add a line for each network card present on the system, for example in our case we'll use two network cards:

The NAT function is in pf (packet filter). To use pf, add the following lines to `/etc/rc.conf`:
```conf
# /etc/rc.conf 

# Hostname
hostname="freebsd"

# WAN connection
ifconfig_xn0="DHCP"

# LAN connection
ifconfig_xn1="inet 192.168.150.1 netmask 255.255.0.0"

# Default gateway
defaultrouter="192.168.133.1" # Set the gateway

# Enable ip forward
gateway_enable="YES"

dhcpd_enable="YES"
dhcpd_ifaces="xn1" # LAN card

pf_enable="YES"                           # Enable PF (load module if required)
pf_rules="/etc/pf.conf"                   # rules definition file for pf
pf_flags=""                               # additional flags for pfctl startup
pflog_enable="YES"                        # start pflogd(8)
pflog_logfile="/var/log/pflog"            # where pflogd should store the logfile
pflog_flags=""                            # additional flags for pflogd startup
pflogd_enable="YES"
pfsync_enable="YES"

# openvpn
openvpn_enable="YES"
openvpn_config="/usr/local/etc/openvpn/openvpn.conf"
```
You have to replace xn0, xn1 with the correct device for your cards, and the addresses with the proper ones.
defaultrouter is needed only if you are not using dhpc for WAN connection.

> NOTE: If you configured the network during installation, some lines about the network cards may be already present. Double check /etc/rc.conf before adding any lines.


You will also have to edit `/etc/hosts` to add the names and the IP addresses of various machines of the LAN, if they are not already there.

```
127.0.0.1        localhost
192.168.150.1    freebsdrouter
```

Set the DNS in `/etc/resolv.conf` :
```
nameserver    192.168.133.1
```

> NOTE: You need to manually set up the nameserver only if you are not using dhcp for WAN connection.

### Set up DHCP Server
The Dynamic Host Configuration Protocol (DHCP) is a network protocol used by hosts (DHCP clients) to retrieve IP address assignments and other configuration information. Each computer connected to a network must have a unique IP, and without DHCP TCP/IP information must be assigned manually on each computer. DHCP Server (or DHCPd) is the server that provides the DHCP client the information it needed.

To install DHCP Server, execute the following commands:
```
# cd /usr/ports/net/isc-dhcp3-server
# make install clean
```

Then edit /usr/local/etc/dhcpd.conf. (local DNS server,lease time, the network gateway, and the available IP address range).
```conf
# /usr/local/etc/dhcpd.conf

# name server
option domain-name-servers 192.168.133.1;
# lease time
default-lease-time 600;
max-lease-time 7200;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;

# Fixed IP address for falcao host using mac address.
host falcao {
    hardware ethernet d0:94:66:a3:c8:34;
    fixed-address 192.168.150.25;
}

# ad-hoc DNS update scheme - set to "none" to disable dynamic DNS updates.
ddns-update-style none;

# Use this to send dhcp log messages to a different log file
# you also have to hack syslog.conf to complete the redirection
log-facility local7;

# subnet declaration
subnet 192.168.0.0 netmask 255.255.0.0 {
range 192.168.150.50 192.168.200.254;
option routers 192.168.150.1;
option subnet-mask 255.255.0.0;
}
```


### Set up FreeBSD Firewall using OpenBSD's PF packet filter

A firewall (in this context) is a set of rules that allows or denies certain types of network packets from entering or leaving your FreeBSD router. This section deals with building a firewall.

We'll used one of the example PF configuration files ` /usr/share/examples/pf/faq-example1` as a basis for our configuration.
This sample file has everything we need for a basic FreeBSD router.
```
# /etc/pf.conf
### pf.conf
### macross
## internal and external interfaces
int_if = "xn1"
ext_if = "xn0"
int_addr = "192.168.150.1"             # Internal IPv4 address (i.e., gateway for private network)
int_network = "192.168.0.0/16"         # Internal IPv4 networkqi

################OPENVPN############################
# default openvpn settings for the client network
vpnclients = "10.8.0.0/24"
# put your tunnel interface here, it is usually tun0
vpnint = "tun0"
# OpenVPN by default runs on udp port 1194
udpopen = "{1194}"
icmptypes = "{echoreq, unreach}"
###################################################

# Ports we want to allow access to from the outside world on our local system (ext_if)
tcp_services = "{ 80 }"

# ping requests
icmp_types = "echoreq"

# Private networks, we are going to block incoming traffic from them
priv_nets = "{ 127.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }"

### options
set block-policy drop
set loginterface $ext_if
set skip on lo0

### Scrub
scrub in all

# NAT traffic from internal network to external network through external interface
nat on $ext_if from $int_if:network to any -> ($ext_if)
nat on $ext_if inet from $vpnclients to any -> $ext_if

### Filters ###

# Permit keep-state packets for UDP and TCP on external interfaces
pass out quick on $ext_if proto udp all keep state
pass out quick on $ext_if proto tcp all modulate state flags S/SA

# Permit any packets from internal network to this host
pass in quick on $int_if inet from $int_network to $int_addr

# Permit established sessions from internal network to any (incl. the Internet)
pass in quick on $int_if inet from $int_network to any keep state
# If you want to limit the number of sessions per NAT, nodes per NAT (simultaneously), and sessions per source IP
#pass in quick on $int_if inet from $int_network to any keep state (max 30000, source-track rule, max-src-nodes 100, max-src-states 500 )

# Permit and log all packets from clients in private network through NAT
pass in quick log on $int_if all

# Block everything
pass in log all
pass out all

pass in on $ext_if proto udp from any to $ext_if port $udpopen 
pass in on $vpnint from any to any
```

Note that this file also defines filter rules which is useful or mandatory to provide the network to users. Eliminate filter section just by this part:
```
pass in all
pass out all
```

Here, we note that file `/var/log/pflog` would be automatically created as follows.
```
# ls -l /var/log/pflog
-rw-------  1 root  wheel  3832 Jul 13 17:45 /var/log/pflog
```


## To check for syntax error
```
# service pf check

OR

/etc/rc.d/pf check

OR

# pfctl -n -f /usr/local/etc/pf.conf
````

## Netstat check
#### xn0
```

# netstat -w1d -I xn0: 
                    input (age0) output 
packets     errs     idrops    bytes      packets        errs    bytes    colls 
5247          0         0       7332276   1600          0     83700      0 
5286          0         0       7331330   1578          0     82296      0 
5278          0         0       7339278   1589          0     83754      0 
5312          0         0       7380344   1570          0     82728      0 
5328          0         0       7337764   1567          0     83160      0 
```
#### xn1
```
# netstat -w1d -I xn1: 
                 input (rl0) output 
packets     errs    idrops     bytes     packets      errs       bytes     colls 
1556          0       0            93508    5133        0       7275788   0 
1547          0       0            92832    5169        0       7337174   0 
1551          0       0            93072    5161        0       7321088   0 
1539          0       0            92352    5199        0       7381268   0 
1520          0       0            91212    5195        0       7367642   0 
```

## Router Status
```
# top –S: 
last pid: 6320; load averages: 0.07, 0.02, 0.00 up 1+18:19:20 10:08:26 
70 processes: 3 running, 55 sleeping, 12 waiting 
CPU: 0.0% user, 0.0% nice, 1.2% system, 4.7% interrupt, 94.2% idle 
Mem: 21M Active, 136M Inact, 89M Wired, 44K Cache, 59M Buf, 237M Free 
Swap: 2048M Total, 2048M Free 
```


## Deploy NGINX with docker
```bash
sudo docker build . -t frc-nginx
sudo docker run --name frc-nginx -d -p 8080:80 -p 80:80 -p 443:443 frc-nginx
```

## Preparing certificates
Setup keys and certificates
```
mkdir /root/sslCA
chmod 700 /root/sslCA
```
This needs to be put in `/etc/ssl/openssl.cnf` as
```
dir            = /root/sslCA    # Where everything is kept
````

OpenSSL also requires some folders and a serial number
```
cd /root/sslCA
mkdir certs private newcerts
echo 1024 > serial
touch index.txt
```

Then, let’s generate the 10 year Certificate Authority which will be used to sign the certificates

```
openssl req -new -x509 -days 3650 -extensions v3_ca -keyout private/cakey.pem -out cacert.pem -config /etc/ssl/openssl.cnf
```
Generate a request for a certificate
```
openssl req -new -nodes -out server.req -keyout private/server.key -config /etc/ssl/openssl.cnf
openssl ca -out server.crt -infiles server.req -config /etc/ssl/openssl.cnf
```
> Note that while you can put almost anything in the fields when asked, you will need at least a common name for each, and the organization must be the same for both certificates. Answer yes [y] where asked.

> If you want to use password-based authentication for the clients, these are all the certificates you need. But if you want certificate based authentication, then each client will also need a certificate. Just create another request and certificate in the same manner, but with “server” replaced by “client”. You will need to copy those files (and the CA cert) to the client later.

```
openssl dhparam -out dh1024.pem 1024
```

## Configure OpenVPN

We based our config on the sample file, so copy it first.
```
cp vpn-server.conf /usr/local/etc/openvpn/openvpn.conf
```

If you want the client to use the VPN as the default gateway, i.e. not only to access server resources, but perhaps also to surf the web, you will need to put these two lines either in the server or client config. You will also need some routing, see below.
```
push "redirect-gateway def1 bypass.dhcp"
push "dhcp-option DNS 8.8.8.8"
```

And then, ignition:
```
/usr/local/etc/rc.d/openvpn starts
```

And then, enable packet forwarding, enable pf as our firewall, and start it
```
sysctl net.inet.ip.forwarding=1
pfctl -ef /etc/pf.conf
```

## Configuring Clients
Install the openvpn client.

Edit the client.ovpn file slightly (it needs to be done as administrator). Look for the following lines and edit accordingly.
```
remote 1.2.3.4 # put your server IP or name here
# the certificate files from the server (separate client certificate, not server)
ca   ca.crt
cert client.crt
key client.key
# remember to keep these lines commented out if they appear
;ns-cert-type server
;remote-cert-tls server
```

At the end add
```
#this line sets the VPN as default gateway
redirect-gateway def1
#these lines stroke Windows 8 the right way
route-method exe
route-delay 5
route-metric 550
```

To start the connection we need to run (as administrator) from the config directory
```
sudo openvpn --config client.ovpn
```
## PF firewall Commands

The commands are as follows. Be careful you might be disconnected from your server over ssh based session:

Start PF
```
# service pf start
````

Stop PF
```
# service pf stop
```

Check PF for syntax error
```
# service pf check
```

Restart PF
```
# service pf restart
```

See PF status
```
# service pf status
```

Show PF rules information
```
# pfctl -s rules
```

Show verbose output for each rule
```
# pfctl -v -s rules
```

How to disable PF from the CLI
```
# pfctl -d
```

How to enable PF from the CLI
```
# pfctl -e
```

How to flush ALL PF rules/nat/tables from the CLI
```
# pfctl -F all
```

## References

https://www.cyberciti.biz/faq/how-to-set-up-a-firewall-with-pf-on-freebsd-to-protect-a-web-server/
https://gundersen.net/openvpn-server-on-freebsd-with-pf-firewall/