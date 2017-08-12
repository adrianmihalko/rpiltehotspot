# Raspberry Pi LTE hotspot on Raspbian Lite

![alt text](images/network_diagram.png "Network diagram")

Recently, I was looking for a solution to share my LTE USB modem connection to the rPi's LAN port. I've searched for hours for a solution with no luck. Others just recommended me to buy an LTE router. Thanks, no. Then a [fantastic guy](https://raspberrypi.stackexchange.com/users/10699/lossleader) on Stackexchange helped me a lot. And the solution was born. In my case I am going to connect rPi LAN port with my Unify USG secondary port to use rPi as a LTE failover connection.

We have 3 connections on the rPi:

1. **WiFi (wlan0)** - main internet connection on rPi. I am going to use this connection to connect to rPi (SSH).
2. **LTE USB modem (wwan0)** - LTE internet connection
3. **LAN (eth0)** - we are going to share wwan0's connection with this interface.

I am not going to show you here how to setup your devices. Please do it before ([WiFi](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md), LTE). This is not a step by step guide. I'll show you the relevant parts of my config files.

`
$cat /etc/network/interfaces


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
`    

From wwan0 config you only need `metric 1000`, `post-up /etc/wwan2lan.sh` lines. Other commands are for setting up internet connection on my Huawei E3372.
