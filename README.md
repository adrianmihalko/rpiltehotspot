# Raspberry Pi LTE hotspot

![alt text](images/network_diagram.png "Network diagram")

Recently, I was looking for a solution to share my LTE USB modem connection to the rPi's LAN port. I've searched for hours for a solution with no luck. Then a fantastic guy[stackexchange](https://raspberrypi.stackexchange.com/users/10699/lossleader) on Stackexchange helped me a lot. In my case I am going to connect rPi LAN port with my Unify USG secondary port to use rPi as a LTE failover connection.

We have 3 connections on the rPi:

1. **WiFi (wlan0)** - main internet connection on rPi. I am going to use this connection to connect to rPi (SSH).
2. **LTE USB modem (wwan0)** - LTE internet connection
3. **LAN (eth0)** - we are going to share wwan0's connection with this interface.

I am not going to show you here how to setup your devices. Please do it before (WiFi, LTE). This is not a step by step guide. I'll show you the relevant parts of my config files.

$cat /etc/network/interfaces

