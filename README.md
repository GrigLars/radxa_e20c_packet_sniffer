# Radxa E20C Packet Sniffer
I built a packet sniffer with a Radxa E20C. Here's how in the short form.

### Background

I bought a really suspicious Wavelink wireless access point, a Wavlink AX3000 Gaming WIFI 6 Router Dual Band for a mere $30 from AliExpress. I was suprised it worked, and actually worked fairly well. Then I was surpsied that it made tens of thosuands of requests to my pi-hole for qqq.com and other things.  WTF was it doing?  Why was it trying to get helper.rumble.network?  Thank goodness I segmented it to a "trash VLAN" for things I am not sure where they came from. I wanted to see what other traffic was going on from a simple WAP besides sktechy DNS queries. 

I had a tiny Radxa E20C with 2 GB NICs I decided to set up.
https://radxa.com/products/network-computer/e20c/

First, I installed their Debian image (not iStoreOS).  Then I set it up via the debug port.  This required the following settings on minicom:
```
pu port /dev/ttyUSB0
pu baudrate 1500000
pu bits 8
pu parity N
pu stopbits 1
pu rtscts No
```
Next, I had to identify the two NICs.  On the outside, they were only labeled WAN and LAN. Through some testing, I determined which was which.
```
# Configure enp1s0 - WAN - The side that faces the router and internet
sudo nmcli connection add type ethernet ifname enp1s0 con-name enp1s0 ipv4.method manual ipv4.addresses 10.0.1.99/16 ipv4.gateway 10.0.1.1
sudo nmcli connection modify enp1s0 ipv4.dns "10.0.1.9"

# Configure eth0 - LAN - the side that faces the device you're sniffing
sudo nmcli connection add type ethernet ifname eth0 con-name eth0 ipv4.method manual ipv4.addresses 10.0.1.101/16 

sudo nmcli connection up enp1s0
sudo nmcli connection up eth0
sudo systemctl restart NetworkManager
```
I connected the WAN nic to my network, and made sure I had IP and DNS.  From here, I had to install some bridge, tcpdump, iptables, and dns utilities:
```
sudo apt install sudo apt-get install bridge-utils dnsutils iptables-persistent tcpdump
```
Then it was time to make a bridge.  Note it has a third IP, which we need to know for later. 
```
# Configure the bridge - br0
sudo nmcli connection add type bridge con-name br0 ifname br0
sudo nmcli connection add type bridge-slave ifname eth0 master br0
sudo nmcli connection add type bridge-slave ifname enp1s0 master br0
sudo nmcli connection up br0

sudo nmcli connection modify br0 ipv4.addresses "10.0.1.100/16" ipv4.gateway "10.0.1.1" ipv4.method manual
sudo nmcli connection up br0
sudo nmcli connection modify br0 ipv4.dns "10.0.1.9"
sudo systemctl restart NetworkManager
```
Now for all the network voodoo.
```
# Configure iptables to forward 
sudo iptables -A FORWARD -i enp1s0 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o enp1s0 -j ACCEPT
sudo apt-get install iptables-persistent
sudo iptables-save > /etc/iptables/rules.v4

# Configure the sysctl
sudo sysctl -w net.ipv4.ip_forward=1
sudo vim /etc/sysctl.conf   # uncomment the ipv4.ip_forward = 1
sudo sysctl -p
```
Now we;'re ready.  I had to do this via the serial console, because the IPs on the interfaces are about to go bye-bye.
```
# Connect the two pieces ot the bridge together - note, this will hang up .99 and .101, so be on serial port?
sudo nmcli con up bridge-slave-enp1s0
sudo nmcli con up bridge-slave-eth0
```
Now SSH into the br0 IP (10.0.1.100 in this example)

### Check your work work - it should look like this
```
radxa@rock-2:~$ nmcli connection show
NAME                 UUID                                  TYPE      DEVICE 
br0                  a9c04ba5-d721-43ba-be55-195fac141e5d  bridge    br0    
bridge-slave-enp1s0  5536bd06-3865-49ae-9202-b99451942379  ethernet  enp1s0 
bridge-slave-eth0    2d4b546c-7908-4a94-9c59-b0db1bb40c87  ethernet  eth0   
enp1s0               9a916819-a84a-4dad-8f34-c36eefbf327b  ethernet  --     
eth0                 0380fc80-e6e7-4ebd-88a6-5d6a8097ba21  ethernet  --     

radxa@rock-2:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether 6a:13:fb:59:f3:0f brd ff:ff:ff:ff:ff:ff
3: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP group default qlen 1000
    link/ether 00:e0:4c:0d:09:66 brd ff:ff:ff:ff:ff:ff
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 32:88:2e:2a:06:df brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.100/16 brd 10.6.255.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 fe80::6a01:d843:c7fe:8ba/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

radxa@rock-2:~$ sudo iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

radxa@rock-2:~$ sudo sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```
Now for the tcpdump part.

```
# 80:3f:5d:xx:xx:xx is the MAC of the WAP
sudo tcpdump -i br0 ether src 80:3f:5d:xx:xx:xx -w ./pcap_file_test1
```

I'll let you know how it goes.
