# OpenVPN on OpenWrt Barrier Breaker

Steps stolen from [Logan Marchione's blog post](http://www.loganmarchione.com/2014/10/openwrt-with-openvpn-client-on-tp-link-tl-mr3020/).

```
ssh root@192.168.1.1
```

Install packages:

```
opkg update
opkg install openvpn-openssl wget unzip
```

Create a new interface for the VPN:

```
cat >> /etc/config/network << EOF
config interface 'PIA_VPN'
    option proto 'none'
    option ifname 'tun0'
EOF
```

Download OpenVPN config from privateinternetaccess.com:

```
cd /etc/openvpn
wget --no-check-certificate https://www.privateinternetaccess.com/openvpn/openvpn.zip
unzip openvpn.zip
rm openvpn.zip
```

Create file with your privateinternetaccess.com credentials:

```
cat >> /etc/openvpn/authuser << EOF
$username
$password
EOF
```

Set permissions on authuser file:

```
chmod 400 /etc/openvpn/authuser
```

Create a generic OpenVPN config:

```
cat >> /etc/openvpn/piageneric.ovpn << EOF
client
dev tun
proto udp
resolv-retry infinite
nobind
persist-key
persist-tun
cipher aes-128-cbc
auth sha1
tls-client
remote-cert-tls server
auth-user-pass authuser
comp-lzo
verb 1
reneg-sec 0
crl-verify crl.rsa.2048.pem
ca ca.rsa.2048.crt
EOF
```

Create firewall zone for new VPN connection:

```
cat >> /etc/config/firewall << EOF
config zone
    option name 'VPN_FW'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option masq '1'
    option mtu_fix '1'
    option network 'PIA_VPN'
 
config forwarding                               
        option dest 'VPN_FW'                    
        option src 'lan' 
EOF
```

Reboot:

```
reboot
```

Log back in:

```
ssh root@192.168.1.1
```

Start the VPN:

```
openvpn --cd /etc/openvpn --config /etc/openvpn/piageneric.ovpn --remote us-siliconvalley.privateinternetaccess.com 1198
```

Confirm that output looks something like this:

```
root@OpenWrt:~# openvpn --cd /etc/openvpn --config /etc/openvpn/piageneric.ovpn --remote us-siliconvalley.privateinternetaccess.com 1198
Sun Nov 20 05:07:49 2016 OpenVPN 2.3.6 mips-openwrt-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [MH] [IPv6] built on Jan  6 2015
Sun Nov 20 05:07:49 2016 library versions: OpenSSL 1.0.2f  28 Jan 2016, LZO 2.08
Sun Nov 20 05:07:49 2016 UDPv4 link local: [undef]
Sun Nov 20 05:07:49 2016 UDPv4 link remote: [AF_INET]104.156.228.162:1198
Sun Nov 20 05:07:49 2016 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Sun Nov 20 05:07:50 2016 [07d5952d3eb922c4cd728b4dd721079f] Peer Connection Initiated with [AF_INET]104.156.228.162:1198
Sun Nov 20 05:07:53 2016 TUN/TAP device tun0 opened
Sun Nov 20 05:07:53 2016 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Sun Nov 20 05:07:53 2016 /sbin/ifconfig tun0 10.7.10.6 pointopoint 10.7.10.5 mtu 1500
Sun Nov 20 05:07:53 2016 Initialization Sequence Completed
```

Check to see if tunnel interface exists (You will have to open a second SSH connection because the openvpn command above must be running):

```
ifconfig tun0
```

```
root@OpenWrt:~# ifconfig tun0
tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
          inet addr:10.132.1.6  P-t-P:10.132.1.5  Mask:255.255.255.255
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:588 errors:0 dropped:0 overruns:0 frame:0
          TX packets:789 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100
          RX bytes:281373 (274.7 KiB)  TX bytes:159631 (155.8 KiB)
```

Close OpenVPN

```
ctrl+c
```

Force router to use privateinternetaccess.com's DNS servers:

```
uci add_list dhcp.lan.dhcp_option="6,209.222.18.222,209.222.18.218"
uci commit dhcp
```

Go to Interfaces - WAN, Advanced Settings, and uncheck "Use DNS servers advertised by peer".  This will show "Use custom DNS servers".  Enter `209.222.18.222` and `209.222.18.218` for privateinternetaccess.  This will help to make sure DNS leak tests do not fail.

Run VPN at startup. Go to Luci web interface, go to System -> Startup and add this before the `exit 0`:

```
openvpn --cd /etc/openvpn --config /etc/openvpn/piageneric.ovpn --remote us-siliconvalley.privateinternetaccess.com 1198 &
```

Reboot for DHCP and startup changes to take effect:

```
reboot
```
