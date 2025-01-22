# The OVN Gateway Router
Building upon my [previous](20160921-An-Introduction-to-OVN-Routing.md) post I will 
now add an OVN gateway router into the lab setup. This gateway router will provide access to the lab network
from within our overlay network.


# The Lab
In order to demonstrate the gateway router we will need to add another physical network to our Ubuntu hosts.
For my purposes I will add the network 10.127.0.128/25 via eth1 of each of my hosts. The final lab diagram
is illustrated below.

![ovn lab](l3Gateway-lab.jpg "ovn lab")


# Introducing the OVN L3 Gateway
An OVN Gateway serves as an onramp/offramp between the overlay network and the physical network. They come
in two flavors: layer-2 which bridge an OVN logical switch into a VLAN, and layer-3 which provide a routed
connection between an OVN router and the physical network. For the purposes of this lab we will 
focus on creating a layer-3 gateway which will serve as the demarcation point between our physical and
logical networks. 

Unlike a distributed logical router (DLR), an OVN gateway router is centralized on a single host (chassis)
so that it may provide services which cannot yet be distributed (NAT, load balancing, etc...). As of
this publication there is a restriction that gateway routers may only connect to other routers via
a logical switch, whereas DLRs may connect to one other directly via a peer link. Work is in progress
to remove this restriction.

It should be noted that it is possible to have multiple gateway routers tied into an environment, which
means that it is possible to perform ingress ECMP routing into logical space. However, it is worth
mentioning that OVN currently does not support egress ECMP between gateway routers. Again, this is
being looked at as a future enhancement.



# Make ubuntu1 an OVN Host
Rather than using a host which houses VMs, lets use ubuntu1 for our OVN gateway router host. Begin by 
registering ubuntu1 with OVN Central (itself):

	ovs-vsctl set open . external-ids:ovn-remote=tcp:127.0.0.1:6642
	ovs-vsctl set open . external-ids:ovn-encap-type=geneve
	ovs-vsctl set open . external-ids:ovn-encap-ip=10.127.0.2

And verify connectivity:

	root@ubuntu1:~# netstat -antp | grep 127.0.0.1
	tcp        0      0 127.0.0.1:6642          127.0.0.1:55566         ESTABLISHED 4999/ovsdb-server
	tcp        0      0 127.0.0.1:55566         127.0.0.1:6642          ESTABLISHED 15212/ovn-controlle

Also, be sure to create the integration bridge if wasn't created automatically by OVN:

	ovs-vsctl list-br
	
	# create if above command had no output
	ovs-vsctl add-br br-int -- set Bridge br-int fail-mode=secure


# OVN Logical Design
Lets review the planned design before we start configuring things. The OVN logical network we are creating
is illustrated below.

![ovn logical components](l3Gateway-logical.jpg "ovn logical components")

As you can see we are adding the following new components:
* OVN gateway router (edge1)
* logical switch (transit) used to connect the edge1 and tenant1 routers
* logical switch (outside) used to connect edge1 to the lab network


# Adding the L3 Gateway
As mentioned earlier the gatway router will be bound to a specific chassis (ubuntu1 in our case).
In order to accomplish this binding we will need to locate the chassis id for ubuntu1. Using the
ovn-sbctl show command from ubuntu1 you should see output similar to this:

	ovn-sbctl show
	Chassis "833ae1bd-ced3-494a-a95b-f2dc54172b71"
			hostname: "ubuntu1"
			Encap geneve
					ip: "10.127.0.2"
					options: {csum="true"}
	Chassis "239f2c28-90ff-468f-a701-655585c630bf"
			hostname: "ubuntu3"
			Encap geneve
					ip: "10.127.0.3"
					options: {csum="true"}
			Port_Binding "dmz-vm2"
			Port_Binding "inside-vm4"
	Chassis "517d558e-158a-4cb2-8870-283e9d39685e"
			hostname: "ubuntu2"
			Encap geneve
					ip: "10.127.0.129"
					options: {csum="true"}
			Port_Binding "inside-vm3"
			Port_Binding "dmz-vm1"

Copy the Chassis UUID of the ubuntu1 host for use below.

Create the new logical router. Be sure to substitute {chassis_id} with a valid UUID. From ubuntu1:

	# create router edge1
	ovn-nbctl create Logical_Router name=edge1 options:chassis={chassis_uuid}

	# create a new logical switch for connecting the edge1 and tenant1 routers
	ovn-nbctl ls-add transit
	
	# edge1 to the transit switch
	ovn-nbctl lrp-add edge1 edge1-transit 02:ac:10:ff:00:01 172.16.255.1/30
	ovn-nbctl lsp-add transit transit-edge1
	ovn-nbctl lsp-set-type transit-edge1 router
	ovn-nbctl lsp-set-addresses transit-edge1 02:ac:10:ff:00:01
	ovn-nbctl lsp-set-options transit-edge1 router-port=edge1-transit
	
	# tenant1 to the transit switch
	ovn-nbctl lrp-add tenant1 tenant1-transit 02:ac:10:ff:00:02 172.16.255.2/30
	ovn-nbctl lsp-add transit transit-tenant1
	ovn-nbctl lsp-set-type transit-tenant1 router
	ovn-nbctl lsp-set-addresses transit-tenant1 02:ac:10:ff:00:02
	ovn-nbctl lsp-set-options transit-tenant1 router-port=tenant1-transit

	# add static routes
	ovn-nbctl lr-route-add edge1 "172.16.255.128/25" 172.16.255.2
	ovn-nbctl lr-route-add tenant1 "0.0.0.0/0" 172.16.255.1
	
	ovn-sbctl show


Notice the port bindings on the ubuntu1 host. You can now test connectivity to edge1
from vm1 on ubuntu2:

	root@ubuntu2:~# ip netns exec vm1 ping 172.16.255.1
	PING 172.16.255.1 (172.16.255.1) 56(84) bytes of data.
	64 bytes from 172.16.255.1: icmp_seq=1 ttl=253 time=1.07 ms
	64 bytes from 172.16.255.1: icmp_seq=2 ttl=253 time=1.13 ms
	64 bytes from 172.16.255.1: icmp_seq=3 ttl=253 time=1.00 ms

           
# Connecting to the "data" Network
We're going to use the eth1 interface of ubuntu1 as our connection point between the edge1 router and the
"data" network. In order to accomplish this we'll need to set up OVN to used the eth1 interface directly
through a dedicated OVS bridge. This type of connection is known as a "localnet" in OVN.

From ubuntu1:

	# create new port on router 'edge1'
	ovn-nbctl lrp-add edge1 edge1-outside 02:0a:7f:00:01:29 10.127.0.129/25

	# create new logical switch and connect it to edge1
	ovn-nbctl ls-add outside
	ovn-nbctl lsp-add outside outside-edge1
	ovn-nbctl lsp-set-type outside-edge1 router
	ovn-nbctl lsp-set-addresses outside-edge1 02:0a:7f:00:01:29
	ovn-nbctl lsp-set-options outside-edge1 router-port=edge1-outside

	# create a bridge for eth1
	ovs-vsctl add-br br-eth1

	# create bridge mapping for eth1. map network name "dataNet" to br-eth1
	ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=dataNet:br-eth1

	# create localnet port on 'outside'. set the network name to "dataNet"
	ovn-nbctl lsp-add outside outside-localnet
	ovn-nbctl lsp-set-addresses outside-localnet unknown
	ovn-nbctl lsp-set-type outside-localnet localnet
	ovn-nbctl lsp-set-options outside-localnet network_name=dataNet

	# connect eth1 to br-eth1
	ovs-vsctl add-port br-eth1 eth1


Test connectivity to edge1-outside from vm1

	root@ubuntu2:~# ip netns exec vm1 ping 10.127.0.129
	PING 10.127.0.129 (10.127.0.129) 56(84) bytes of data.
	64 bytes from 10.127.0.129: icmp_seq=1 ttl=253 time=1.74 ms
	64 bytes from 10.127.0.129: icmp_seq=2 ttl=253 time=0.781 ms
	64 bytes from 10.127.0.129: icmp_seq=3 ttl=253 time=0.582 ms



# Giving the Ubuntu Hosts Access to the "data" Network
Lets give the Ubuntu hosts a presence on the data network. For ubuntu2/ubuntu3 its simply a matter of setting
an IP on their physical nics (eth1 in my setup). For ubuntu1 we'll set an IP on the br-eth1 interface.

On ubuntu1:

	ip addr add 10.127.0.130/24 dev br-eth1
	ip link set br-eth1 up

On ubuntu2:

	ip addr add 10.127.0.131/24 dev eth1
	ip link set eth1 up
	
On ubuntu3:

	ip addr add 10.127.0.132/24 dev eth1
	ip link set eth1 up

Testing from ubuntu1 to edge1

	root@ubuntu1:~# ping 10.127.0.129  
	PING 10.127.0.129 (10.127.0.129) 56(84) bytes of data.
	64 bytes from 10.127.0.129: icmp_seq=1 ttl=254 time=0.563 ms
	64 bytes from 10.127.0.129: icmp_seq=2 ttl=254 time=0.290 ms
	64 bytes from 10.127.0.129: icmp_seq=3 ttl=254 time=0.333 ms



# Configuring NAT
Lets see what happens when we attempt to ping ubuntu1 from vm1:

	root@ubuntu2:~# ip netns exec vm1 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	^C
	--- 10.127.0.130 ping statistics ---
	3 packets transmitted, 0 received, 100% packet loss, time 2016ms


Unsurprisingly, this does not work. Why not? Lets look at the output of a tcpdump from ubuntu1:

	root@ubuntu1:~# tcpdump -i br-eth1
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on br-eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
	14:41:53.057993 IP 172.16.255.130 > 10.127.0.130: ICMP echo request, id 19359, seq 1, length 64
	14:41:54.065696 IP 172.16.255.130 > 10.127.0.130: ICMP echo request, id 19359, seq 2, length 64
	14:41:55.073667 IP 172.16.255.130 > 10.127.0.130: ICMP echo request, id 19359, seq 3, length 64


We can see the requests coming in, however our responses are returning through a different interface (not seen in
the tcpdump output). This is due to the fact that ubuntu1 has no route to 172.16.255.130 and is responding
via its own default gateway. In order to get things working we will need to take 1 of 2 possible approaches:
1. add static routes on the Ubuntu hosts
2. set up NAT on the OVN gateway router

We'll opt for option 2 because it is much less of a hassle than trying to manage static routes.

With OVN there are 3 types of NAT which may be configured:
* DNAT -- used to translate requests to an externally visible IP to an internal IP
* SNAT -- used to translate requests from one or more internal IPs to an externally visible IP
* SNAT-DNAT -- used to create a "static NAT" where an external IP is mapped to an internal IP, and vice versa

Since we don't need (or want) the public network to be able to directly access our internal VMs, lets focus on allowing
outbound SNAT from our VMs. In order to create NAT rules we'll need to manipulate the OVN northbound database directly. The
syntax is a bit strange but I'll explain later. From ubuntu1:

	# create snat rule which will nat to the edge1-outside interface
	ovn-nbctl -- --id=@nat create nat type="snat" logical_ip=172.16.255.128/25 \
	external_ip=10.127.0.129 -- add logical_router edge1 nat @nat


In brief, this command is creating an entry in the "nat" table of the northbound database, storing the resulting UUID
within the ovsdb variable "@nat", and then adding the UUID stored in @nat to the "nat" field of the "edge1" entry
in the "logical_router" table of the northbound database. If you want to know the details of the northbound database
then be sure to check out the man page for ovn-nb. The man page for ovn-nbctl also explains the command syntax used above.

Testing connectivity from vm1:

	root@ubuntu2:~# ip netns exec vm1 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	64 bytes from 10.127.0.130: icmp_seq=40 ttl=62 time=2.39 ms
	64 bytes from 10.127.0.130: icmp_seq=41 ttl=62 time=1.61 ms
	64 bytes from 10.127.0.130: icmp_seq=42 ttl=62 time=1.28 ms


As seen above, we can now ping the outside world from our internal VMs.


# Final Words
Overlay networks are almost entirely useless unless you can connect them to the outside world. OVN gateways provide a means
for making such a connection. 

In the [next](http://blog.spinhirne.com/2016/09/the-ovn-load-balancer.html) post I will explore another important feature
of OVN: the OVN load balancer.
