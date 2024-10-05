# Router machine

```
sudo gedit /etc/sysctl.conf  
net.ipv4.ip_forward = 1  
sudo sysctl -p
```


`sudo iptables -t nat -A POSTROUTING -o wlx0013eff30e78 -j MASQUERADE`

* `-A POSTROUTING` specifies that the rule applies to packets after they have been routed.  
* `-o wlx0013eff30e78` specifies the outgoing interface, which is your Wi-Fi interface connected to the internet.  
* `-j MASQUERADE` tells the system to replace the source IP address of outgoing packets with the IP address of the interface (in this case, the public IP of `wlx0013eff30e78`). This way, the return packets can find their way back to the originating device.

`sudo iptables -A FORWARD -i wlx0013eff30e78 -o eth0 -j ACCEPT`

* `-A FORWARD` indicates that this rule applies to packets being forwarded.  
* `-i wlx0013eff30e78` specifies that the packets are coming in from the Wi-Fi interface.  
* `-o enp0s31f6` specifies that the packets are going out to the Ethernet interface connected to the other machine.  
* `-j ACCEPT` allows these packets to be forwarded.

`sudo iptables -A FORWARD -i enp0s31f6 -o wlx0013eff30e78 -m state --state RELATED,ESTABLISHED -j ACCEPT`

* `-m state --state RELATED,ESTABLISHED` matches packets that are related to existing connections (i.e., responses to requests made by the devices on the local network).  
* This is crucial for allowing replies from the internet back to the local devices.

Wlx0013eff30e78 is a wifi internet interface. Enp0s31f6 is connected to another machine’s enp0s31f6. They have manual IP 192.168.1.102 and 101 respectively. 

# To persist 
We have two options: (1) Using iptables-persistent (2) Using script

## Using iptables-persistent
```
sudo apt install iptables-persistent
sudo iptables -t nat -A POSTROUTING -o wlx0013eff30e78 -j MASQUERADE
sudo iptables -A FORWARD -i wlx0013eff30e78 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i enp0s31f6 -o wlx0013eff30e78 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo netfilter-persistent save
```

## Using script
```
sudo nano /etc/network/if-up.d/router-iptables
#!/bin/sh
  iptables -t nat -A POSTROUTING -o wlx0013eff30e78 -j MASQUERADE
  iptables -A FORWARD -i wlx0013eff30e78 -o eth0 -j ACCEPT
  iptables -A FORWARD -i enp0s31f6 -o wlx0013eff30e78 -m state --state RELATED,ESTABLISHED -j ACCEPT
```
`sudo chmod +x /etc/network/if-up.d/router-iptables`

To check
```
sudo iptables -L
sudo iptables -t nat -L
```

# Other machine

```
sudo ip route add default via 192.168.1.102  
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf  
nameserver 8.8.8.8
```


