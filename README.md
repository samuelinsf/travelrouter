# travelrouter

When traveling, it would be very nice to be able to VPN into my home network.
It would be great if my traveling devices appeared to be attached to the
internet from my home router, instead of whatever distant land I happen to be in.

I would like this to work across all my devices.
Laptop, phone, tablet, and my family's devices as well.
All of these devices easily connect via wifi, so why not use that
path to put them all on a secure VPN back to my home network?

I couldn't find a good solution to this, so here are some notes on how I built my own.

Features:

* Small hardware (box of playing cards sized, lightweight, low power, battery optional)
* Provides a secure wifi network, devices connected to that network appear to be connected to the internet via my home ISP connection.
* Traveling vpn device uses wifi client mode (or ethernet) to connect to the untrusted network.
* System fails safe, if the VPN cannot come up, my devices never talk on the untrusted network.
* Supports ethernet bridge mode, so all protocols and technologies work over the VPN.
* All based on Openwrt (2015 chaos calmer version) and Quicktun (a nacl crypto simple vpn)


These notes and hints were prepared several months after I quickly setup a vpn router for portable use.
There are probably some details missing, but hopefully this is a start for you and your needs.

This attempts to document the build of my setup for a traveling gl inet 6416 which offers a wifi AP which is tunneled to my home router. 

* See: http://www.gl-inet.com/gl-inet6416/
* And on amazon: https://www.amazon.com/Gl-iNet-Smart-Router-Openwrt-Modem/dp/B00JKFE0FW

All this is setup on openwrt chaos calmer aka 15.05.

## Quicktun install.

To build quicktun I did these steps (OpenWRT docs may provide a quicker method, this is what worked for me).
(OpenWRT docs: https://wiki.openwrt.org/doc/howto/build has build prerequisites .. I already had these on my system)

```
git clone git://git.openwrt.org/15.05/openwrt.git
cd openwrt
git checkout 15.05
```

Add the nacl library with the feeds system:

```
./scripts/feeds update -a
./scripts/feeds install nacl
```

Then copy the contents of the `quicktun` directory from this repo into `openwrt/package/quicktun`.

`make menuconfig`

Set the `Target System` to your platform, in my case `Atheros AR7xxx/AR9xxx`.

Set quicktun to be built, go to `Network -> VPN` and for the quicktun package set it to be built `<*>`.

Set nacl to be built, go to `Libraries` and for nacl package set to be built: `<*>`.

Exit menuconfig, saving your changes to .config.
Then you can do a `make`.

Find the package file .. I suppose there is a right way to do this .. but I just ran:
```
find . -name '*quicktun*ipk'
```
On my system the package file showed up in: `bin/ar71xx/packages/base/quicktun_2.2.5-4_ar71xx.ipk`


I built the package on my intel desktop linux system, and copied it over to the home, and roving routers, then did:
```
router$ opkg install quicktun_2.2.5-4_ar71xx.ipk
```

Then I made two sets of keys, one for the home router, one for the roving router.
```
quicktun.keypair
...
quicktun.keypair
```

Save these, they go into the /etc/config/quicktun files on the hosts (see the comments in the config for clarity on which key strings go where).

## Roving router.

On the roving router there is a bridge network with the vpn'd wifi AP on it, and the tap0 from the quicktun. In this example the network is named quicktun0:

```
From roving /etc/config/network:
...
config interface 'quicktun0'
    option proto 'static'
    option type 'bridge'
    option _orig_ifname 'tap0'
    option _orig_bridge 'true'
    option netmask '255.255.255.0'
    option ipaddr '10.0.42.1'
    option mtu '1480'
    option ifname 'tap0'

...

```

The home side is the default route (in this example 10.0.42.4 ). And we use 8.8.8.8 as the name server (which goes out through the masq on the home side. But we run dhcp on the roving router, so clients are always assigned default route config, mtu and IP numbers.

```
From roving /etc/config/dhcp:

...
config dhcp 'quicktun0'
    option start '100'
    option leasetime '12h'
    option limit '150'
    option interface 'quicktun0'
    # set default route
    list dhcp_option '3,10.0.42.4'
    list dhcp_option '6,8.8.8.8'
    # set mtu
    list dhcp_option '26,1480'

...
```

The firewall setup needs holes for the udp used by quicktun.

```
From roving /etc/config/firewall:

...
config rule
    option target 'ACCEPT'
    option src 'wan'
    option proto 'udp'
    option dest_port '2777'
    option name 'allow quick tun'

config zone
    option input 'ACCEPT'
    option forward 'REJECT'
    option output 'ACCEPT'
    option name 'quicktun0'
    option network 'quicktun0'

config rule
    option target 'ACCEPT'
    option proto 'tcp'
    option dest_port '22'
    option name 'allow ssh'
    option src 'quicktun0'

...
```

The wireless AP is attached to the network with tap0 from quicktun:

```
From roving /etc/config/wireless:

...
config wifi-iface
    option device 'radio0'
    option mode 'ap'
    option ssid 'safe wifi'
    option network 'quicktun0'
    option encryption 'psk-mixed'
    option key 'wifi psk secret key'

...
```

Quicktun on the roving router provides a tap0 hooked to a tap0 on the home router.

```
From the roving /etc/config/quicktun:
(I deleted the default configuration.)
...

config quicktun tap0
    option enabled 1
    option local_port 2777
    # dns name or ip are both ok here
    option remote_address dns.name.of.your.home.router.dyndns.org
    option remote_port 2777
    option remote_float 1
    option protocol "salty"
    option tun_mode 0
    option interface "tap0"

    # The local private key and the remote public key
    # A keypair can be generated with quicktun.keypair
    option private_key 0000000000000000000000000000000000000000000000000000000000000000
    option public_key 0000000000000000000000000000000000000000000000000000000000000000

...
```

# Home side.

My home router is an engenius ar1750, running openwrt, but a glinet 6416 or any other openwrt you can get the quicktun package built for can work. The ar1750 is object code compatible with the glinet 6416, making things easy.

Quicktun creates a tap0 which is setup to masquerade to the open internet.

```
From home side /etc/config/network:

...
config interface 'quicktun0'
    option proto 'static'
    option ifname 'tap0'
    option ipaddr '10.0.42.4'
    option netmask '255.255.255.0'
    option type 'bridge'
    option mtu '1480'

...
```

The quicktun0 network is masqueraded over to the wan. Port 2777 is allowed in from the wan side.

```
From home side /etc/config/firewall:

...
config rule              
        option target 'ACCEPT'
        option src 'wan'
        option proto 'udp'            
        option dest_port '2777'
        option name 'quicktun'


config forwarding        
        option dest 'wan'    
        option src 'quicktun0'

...
```

Quicktun on the home side is configured for the roaming unit to be at any ip.

```

From home side /etc/config/quicktun:
(I deleted the default configuration.)

...
config quicktun tap0
    option enabled 1
    option local_port 2777
    option remote_float 1
    option protocol salty
    option tun_mode 0
    option interface "tap0"
    # The local private key and the remote public key
    # A keypair can be generated with quicktun.keypair
    option private_key 0000000000000000000000000000000000000000000000000000000000000000
    option public_key 0000000000000000000000000000000000000000000000000000000000000000

...

```
## Known Problems

If the travel router is configured in wifi client mode, and AP mode at the same time, and the client mode wifi becomes unavailable, the AP wifi will stop working. The fix is to disable the client wifi connection. I typically do this in the web UI before I leave a hotel where the router is on the wifi as a client. If I forget I have to connect to the travel router over the ethernet interface to fix the router.  This bug is in the hostapd for openwrt. See https://dev.openwrt.org/ticket/12000#comment:9  and https://forum.openwrt.org/viewtopic.php?id=41610 for more information about this. Ideally we should just debug and fix hostapd. (Its not that bad, just need to fully grok the netlink situation it is getting into).

Sometimes quicktun fails to come up if dns is not available on the wan at the right time.  Running `/etc/init.d/quicktun restart` fixes this.

## Captive portal.

When the roving router's wan link is a captive portal, a socks proxy running on the roving router can allow a browser on the safe wifi network to interact with the portal, thus allowing a safe path to registering and getting the captive portal to pass the vpn traffic.

```
Install sockd:

$ opkg install sockd

The sockd setup:

/etc/sockd.conf:

timeout.io: 30
internal: 10.0.42.1 port=5555
external: 192.168.8.1
child.maxidle: yes
clientmethod: none
method: none
client pass {
        #from: 192.0.2.0/24 to: 0.0.0.0/0
        from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error # connect disconnect
}
pass {  
        from: 0.0.0.0/0 to: 0.0.0.0/0
        command: bind connect udpassociate
        log: error # connect disconnect iooperation
}
pass {
        from: 0.0.0.0/0 to: 0.0.0.0/0
        command: bindreply udpreply
        log: error # connect disconnect iooperation
}


Add firewall rules for socks:

From roaming /etc/config/firewall:

...
config rule
        option enabled '1'            
        option target 'ACCEPT'
        option proto 'tcp'
        option dest_port '5555'
        option name 'allow socks5'
        option src 'lan'    

...

Append startup for sockd in /etc/rc.local:
sockd -D

The roaming router is using the dhcp server's resolver:
verify /etc/resolv.conf is a symlink pointing to /tmp/resolv.conf.auto

Start firefox on OSX with a special profile to use the socks5 path:

OSX$ /Applications/Firefox.app/Contents/MacOS/firefox --profile socks-on-router

You may need to login to the roaming router and restart sockd after the wan  is stable:
$ killall sockd ; sockd -D

```

## Fixing the Build on OpenWRT

I found that during this process I needed to fix the quicktun build package a lot.
This command was handy to rebuild the package only from the openwrt build root:
```
make package/quicktun/clean V=s
make package/quicktun/compile V=s
```

I also found the openwrt package docs to be helpful:
https://wiki.openwrt.org/doc/devel/packages
https://wiki.openwrt.org/inbox/procd-init-scripts
