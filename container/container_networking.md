## Replicating container networking
#command to create network namespace

$ sudo ip netns add netns0

#command to list the container\
$ ip netns

#enter network namespace

$ sudo nsenter --net=/var/run/netns/netns0 bash

#creating a pair of veth devices
(veth devices comes in pair. They create tunnel between two network namespace. Data packet send to one pair will be automatically delivered to second pair.)

$ sudo ip link add veth0 type veth peer name ceth0

# moving one end of veth pair device to namespace netns0
(By default both veth pair device reside into root namespace(host namespace). For connect root namespace with newly created network namespace netns0. We have to move one end of veth pair to the netns0).

$ sudo ip link set ceth0 netns netns0

# setting the network device up and assigning a ip address to it

---- inside root namespace ----------
$ sudo ip link set veth0 up
$ sudo ip addr add 172.18.0.11/16 dev veth0

---- inside netns0 namespace ----------
$ sudo nsenter --net=/var/run/netns/netns0
$ ip link set lo up
$ sudo ip link set ceth0 up
$ sudo ip addr add 172.18.0.10/16 dev ceth0


-> In the above approach we created the a new network namespace netns0. We also created a pair of veth devices veth0 and ceth0 and then added ceth0 to new namespace netsn0. After all this we assigned the ipaddress to the veth devices. Finally we were able to reach the root namespace from the netns0 namespace and vice versa.

-> The problem came when we created one more namespace netns1. Plus we also created one more pair of veth devices veth1 and ceth1,then we added the ceth1 to namespace netns1. After this we assigned the ip  to veth1 and ceth1 devices in the same network segment(172.18.0.0/16) as veth0 and ceth0. when we tried reaching root namespace from netns1, it failed to reach and the same is true for vice versa. However we were able to reach the veth1 from the netns0 namespace.

-> This is happening because we are using same ip sengment for both netns1 and netns0 namespace, which is a valid case.
-> So Now we will be taking the help of network bridge(virtual network switch) to overcome the problem. Bridge is a virtual network switch which works on layer 2(ethernet layer).

# Replicating container networking using network bridge

-> First create a network namespace netns1
-> Create a pair of veth devices veth0 and ceth0
-> Move ceth0 to network namespace netns0 and assign it ip address 172.18.0.10/16.
-> Now again create a second network namespace netns1
-> Create a pair of veth devices veth1 and ceth1.
-> Move ceth1  to newly craeted namespace netns1. Assign ip address 172.18.0.20/16 to device ceth1

# Create bridge interface
$ ip link add br0 type bridge
$ ip link set br0 up

# Attaching devices veth0 and veth1 to bridge br0
$ sudo ip link set veth0 master br0
$ sudo ip link set veth1 master br0

-> Now if you ping ceth1 from network namespace netns0. It will be successful. Same is true for vice versa. So we have been 
able to connect the two devices from two different namespace sitting in same network segment with the help of network bridge.

Congratulation......

# making connectivity to root namespace
-> Our network namespace can reach to each other. However they can not talk to root namespace.
-> To establish the connectivity between the root and container network namespace, we have to assign the ip address to bridge
br0. Once we do this one route will be added to the host.

$ ip addr add 172.18.0.1/16 dev br0

-> After assigning the ip to the bridge. we are able to ping container namespace from root namespace. However we are still not able to ping the root namespace from netns0 and netns1. For enabling the connectivity from container namespace to root 
namespace we have to add the default route inside container namespace.

# Adding default route inside container network namespace.
--- run the below command into both container namespace ---
$ ip route add default via 172.18.0.1

-> Now we are able to ping the root namespace from contaienr namespace.
-> If you trying ping 8.8.8.8 . It still hung. So our container namepspace is still not connected with outside world.

# Enable packet forwarding on host
Our host will now work as router for container and bridge will work as gateway.
$ sudo bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'

-> The reason our container namespace is not able to connect with outside world. Cause outside world server would not able to
send packet back to the containeri namespace, since they have private ip. Routing to container ip is only known local network.

->To overcome above proble we have to take the help of NAT(Netwrk address translation).When any packet we send to ourside world from container namespace. Container ip will be replaced by host ip and when response will come to host. Host will again replace the ip to container ip.
-> For above we have to add some rule to Post routing chain of NAT table.

# Enabling Nat for container network segment accept packet coming from bridge
$ sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -o br0 -j MASQUERADE

-> Now try pinging 8.8.8.8. it will work.

# Making container rechable from outside world(Port Publishing).

---------- commmand to run the server -----
$ python3 -m http.server --bind 172.18.0.10 5000

-> Run the server inside one of container network namespace. Then try reaching to it from host, we will get a respose since 
host is aware about how to route to the container namespace.

-> However of we try reaching to server from another machine. First question will come to mind on which ip we should make the
request. since container ip is private and not visible to the outside world. So let's make request to the host ip 10.0.2.15   and port 5000. We will not get any response. Therefore as of now our container namespace is not rechable from outside of host.

-> Since outside world can see the host ip not the container ip. We have to publish the container port on some host port. That means somehow we have to map the container port to a specific host port. And whenever a packet will come to host port mapped to container port. We will forward the packet to conatiner port. This way even the outside world will able to access the our  contaner.

----- We will be using following iptables command to do the port forwarding --------- 
External traffic 

$ sudo iptables -t nat -A PREROUTING -d 10.0.2.15 -p tcp -m tcp --dport 5000 -j DNAT --to-destination 172.18.0.10:5000

Local traffic (since it does not pass the PREROUTING chain)
$ sudo iptables -t nat -A OUTPUT -d 10.0.2.15 -p tcp -m tcp --dport 5000 -j DNAT --to-destination 172.18.0.10:5000

-> After this if you will try ping 10.0.2.15:5000, you will get the response.
