# Raspberry Pi LTE hotspot on Raspbian Lite

![alt text](images/network_diagram.png "Network diagram")

Recently, I was looking for a solution to share my LTE USB modem connection to the rPi's LAN port. I've searched for hours for a solution with no luck. Others just recommended me to buy an LTE router. Thanks, no. Then a [fantastic guy](https://raspberrypi.stackexchange.com/users/10699/lossleader) on Stackexchange helped me a lot. And the solution was born. In my case I am going to connect rPi LAN port with my Unify USG secondary port to use rPi as a LTE failover connection.

We have 3 connections on the rPi:

1. **WiFi (wlan0)** - main internet connection on rPi. I am going to use this connection to connect to rPi (SSH).
2. **LTE USB modem (wwan0)** - LTE internet connection
3. **LAN (eth0)** - we are going to share wwan0's connection with this interface.

I am not going to show you here how to setup your devices. Please do it before ([WiFi](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md), LTE). This is not a step by step guide. I'll show you the relevant parts of my config files.

First, install dnsmasq package:

    sudo apt-get install dnsmasq

/etc/network/interfaces:
        
    #please use different subnet as your wlan0
    #eth0:
    auto eth0  
    iface eth0 inet static  
    	address 192.168.2.1
    	netmask 255.255.255.0
    
    #wlan0:
    allow-hotplug wlan0
    iface wlan0 inet manual
        wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
        metric 0  
    
    #wwan0:
    allow-hotplug wwan0
    iface wwan0 inet dhcp
        metric 1000
        pre-up /bin/sleep 10
        pre-up /bin/echo -e "AT^NDISDUP=1,1,\"internet\"\r" > /dev/ttyUSB0
        post-up /etc/wwan2lan.sh
        post-down /bin/echo -ne 'AT^NDISDUP=1,0\r\n' > /dev/ttyUSB


From wwan0 config you only need `metric 1000`, `post-up /etc/wwan2lan.sh` lines. Other commands are for setting up internet connection on my Huawei E3372.

Turn on NAT IPV4 forwarding in /etc/sysctl.conf:

    $cat /etc/sysctl.conf
    net.ipv4.ip_forward=1

Setup internet connection sharing in Iptables:

    sudo iptables -t nat -A POSTROUTING -o wwan0 -j MASQUERADE

Configure it to load on reboot by first saving it to a file:

    sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"

Then create a 'hook' file with a line to restore the ip tables :

    sudo nano /lib/dhcpcd/dhcpcd-hooks/70-ipv4-nat

Add:

iptables-restore < /etc/iptables.ipv4.nat


The following script makes the real deal, so it's important:

/etc/wwan2lan.sh

    #!/bin/bash
   
    sleep 10
    
    # define interfaces
    
    LOCALIF="wlan0"
    WWANIF="wwan0"
    LANIF="eth0"
    
    #get gateway IP address
    
    LOCALGATEWAY=$(ip route show 0.0.0.0/0 dev $LOCALIF | cut -d\  -f3)
    WWANGATEWAY=$(ip route show 0.0.0.0/0 dev $WWANIF | cut -d\  -f3)
     
    #local traffic gets one private default route:
    
    ip rule add iif lo priority 48000 table 2
    ip route add default via $LOCALGATEWAY dev $LOCALIF table 2
    
    #eth0 traffic gets another private default route:
    
    ip rule add iif $LANIF priority 48010 table 3
    ip route add default via $WWANGATEWAY dev $WWANIF table 3
    
    #delete shared gateways
    
    ip route delete default dev $LOCALIF
    ip route delete default dev $WWANIF
    
    #enable wwan0 traffic from localhost if we specify the adapter (example: curl --interface wwan0 google.com)
    INETADDR=$(/sbin/ifconfig $WWANIF | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}')
    ip rule add from $INETADDR/32 iif lo priority 47010 table 3

Setup Dnsmasq, if you used different subnet for eth0, you must change it here as well:

/etc/dnsmasq.conf

    interface=eth0      # Use interface eth0  
    listen-address=192.168.2.1 # Explicitly specify the address to listen on  
    #bind-interfaces      # Bind to the interface to make sure we aren't sending things elsewhere  
    #server=8.8.8.8       # Forward DNS requests to Google DNS  
    domain-needed        # Don't forward short names  
    #bogus-priv           # Never forward addresses in the non-routed address spaces.  
    dhcp-range=192.168.2.1,192.168.2.50,12h # Assign IP addresses between 172.24.1.50 and 172.24.1.150 with a 12 hour lease time  
    log-queries
    
    
