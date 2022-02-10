# Networking
## Networking pre-requisites
### Switching and Routing
- Switching
- Routing
A router helps connect two networks together.
- Default Gateway
If the network was a room, the gateway is a door to the outside world to other network or to the internet.A
``` bash
route
ip route add 192.168.2.0/24 via 192.168.1.1
vim /etc/network/interfaces
cat /proc/sys/net/ipv4/ip_forward # check if IP forwarding is enable on a host
1
```
### DNS
- DNS configurations on linux
- CoreDNS introduction
### Network namespaces
### Docker Networking 
### Linux interfaces for virtual networking
- Bridge Network
- VLAN
- VXLAN
