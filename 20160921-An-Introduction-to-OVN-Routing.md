
# An Introduction to OVN Routing
Building upon my [previous](20160919-A-Primer-On-OVN.md) post I will now add
basic layer-3 networking into the OVN setup. The end result will be a pair of logical switches connected
by a logical router. As an added bonus, the router will be configured to serve IP addresses via 
the DHCP service which is built into OVN.


# Re-Architecting the Logical Components
Since the setup is starting to become more complex we're going to re-architect a bit. The new setup will
be as follows:
* 2 logical switches: "dmz" and "inside"
* logical router "tenant1" connecting the two logical switches
* IP network 172.16.255.128/26 for "dmz"
* IP network 172.16.255.192/26 for "inside"
* a pair of "VMs" on each logical switch

The proposed logical network is illustrated below.

![ovn logical components](routing-intro-logical.jpg "ovn logical components")



# A Word On Routing
As part of our setup we will be creating an OVN router, also known as a "distributed logical router" (DLR). 
A DLR differs from a traditional router in that it is not an actual appliance but rather a 
logical construct (not unlike a logical switch). DLRs exist solely as a function in OVS:
in other words each OVS instance is capable of simulating a layer-3 router hop locally prior to forwarding
the traffic across the overlay network. 


# Creating the Logical Switches and Router
Define the new logical switches. From ubuntu1:

	ovn-nbctl ls-add inside
	ovn-nbctl ls-add dmz


Add the logical router with its associated router and switch ports:

	# add the router
	ovn-nbctl lr-add tenant1
	
	# create router port for the connection to dmz
	ovn-nbctl lrp-add tenant1 tenant1-dmz 02:ac:10:ff:01:29 172.16.255.129/26
	
	# create the dmz switch port for connection to tenant1
	ovn-nbctl lsp-add dmz dmz-tenant1
	ovn-nbctl lsp-set-type dmz-tenant1 router
	ovn-nbctl lsp-set-addresses dmz-tenant1 02:ac:10:ff:01:29
	ovn-nbctl lsp-set-options dmz-tenant1 router-port=tenant1-dmz
	
	# create router port for the connection to inside
	ovn-nbctl lrp-add tenant1 tenant1-inside 02:ac:10:ff:01:93 172.16.255.193/26
	
	# create the inside switch port for connection to tenant1
	ovn-nbctl lsp-add inside inside-tenant1
	ovn-nbctl lsp-set-type inside-tenant1 router
	ovn-nbctl lsp-set-addresses inside-tenant1 02:ac:10:ff:01:93
	ovn-nbctl lsp-set-options inside-tenant1 router-port=tenant1-inside
	
	ovn-nbctl show



# Adding DHCP
DHCP within OVN works a bit differently than most server solutions. The general idea is that the administrator
will:
1. define a set of DHCP options for use with a given subnet
2. create logical switch ports which define both the mac address and the IP address expected to exist behind that port
3. assign the DHCP options to that port.
4. set port security to allow only the assigned addresses

Lets get started by configuring the logical ports for the (4) VMs we'll be adding. From ubuntu1:

	ovn-nbctl lsp-add dmz dmz-vm1
	ovn-nbctl lsp-set-addresses dmz-vm1 "02:ac:10:ff:01:30 172.16.255.130"
	ovn-nbctl lsp-set-port-security dmz-vm1 "02:ac:10:ff:01:30 172.16.255.130"
	
	ovn-nbctl lsp-add dmz dmz-vm2
	ovn-nbctl lsp-set-addresses dmz-vm2 "02:ac:10:ff:01:31 172.16.255.131"
	ovn-nbctl lsp-set-port-security dmz-vm2 "02:ac:10:ff:01:31 172.16.255.131"
	
	ovn-nbctl lsp-add inside inside-vm3
	ovn-nbctl lsp-set-addresses inside-vm3 "02:ac:10:ff:01:94 172.16.255.194"
	ovn-nbctl lsp-set-port-security inside-vm3 "02:ac:10:ff:01:94 172.16.255.194"
	
	ovn-nbctl lsp-add inside inside-vm4
	ovn-nbctl lsp-set-addresses inside-vm4 "02:ac:10:ff:01:95 172.16.255.195"
	ovn-nbctl lsp-set-port-security inside-vm4 "02:ac:10:ff:01:95 172.16.255.195"
	
	ovn-nbctl show

You may have noted that, unlike the previous lab, we are defining both mac and IP addresses as part of the logical
switch definition. The IP address definition serves 2 purposes for us: 
1. It enables ARP suppression by allowing OVN to locally answer ARP requests for IP/mac combinations it knows about.
2. It acts as a DHCP host assignment mechanism by issuing the defined IP address to any DHCP requests it sees from that port.

Next we need to define our DHCP options and assign them to our logical ports. The process here is going to be a bit different
than we've seen before in that we will be directly interacting with the OVN NB database. The reason for this approach is
that we need to capture the UUID of the DHCP_Options entry we create so that we can assign it to our switch ports. To do
this we will capture the output of the ovn-nbctl command to a pair of bash variables. 

	dmzDhcp="$(ovn-nbctl create DHCP_Options cidr=172.16.255.128/26 \
	options="\"server_id\"=\"172.16.255.129\" \"server_mac\"=\"02:ac:10:ff:01:29\" \
	\"lease_time\"=\"3600\" \"router\"=\"172.16.255.129\"")" 
	echo $dmzDhcp

	insideDhcp="$(ovn-nbctl create DHCP_Options cidr=172.16.255.192/26 \
	options="\"server_id\"=\"172.16.255.193\" \"server_mac\"=\"02:ac:10:ff:01:93\" \
	\"lease_time\"=\"3600\" \"router\"=\"172.16.255.193\"")"
	echo $insideDhcp
	
	ovn-nbctl dhcp-options-list


See the man page for ovn-nb if you want to know more about the OVN NB database.

Now we'll assign the DHCP_Options to our logical switch ports using the UUID stored within the variables.

	ovn-nbctl lsp-set-dhcpv4-options dmz-vm1 $dmzDhcp
	ovn-nbctl lsp-get-dhcpv4-options dmz-vm1
	
	ovn-nbctl lsp-set-dhcpv4-options dmz-vm2 $dmzDhcp
	ovn-nbctl lsp-get-dhcpv4-options dmz-vm2
	
	ovn-nbctl lsp-set-dhcpv4-options inside-vm3 $insideDhcp
	ovn-nbctl lsp-get-dhcpv4-options inside-vm3
	
	ovn-nbctl lsp-set-dhcpv4-options inside-vm4 $insideDhcp
	ovn-nbctl lsp-get-dhcpv4-options inside-vm4



# Configuring the VMs
As in the last lab we will be using fake "VMs" using OVS internal ports and network namespaces. The difference now
is that we will use DHCP for address assignments. Lets set up the VMs.

On ubuntu2:

	ip netns add vm1
	ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
	ip link set vm1 address 02:ac:10:ff:01:30
	ip link set vm1 netns vm1
	ovs-vsctl set Interface vm1 external_ids:iface-id=dmz-vm1
	ip netns exec vm1 dhclient vm1
	ip netns exec vm1 ip addr show vm1
	ip netns exec vm1 ip route show
	
	ip netns add vm3
	ovs-vsctl add-port br-int vm3 -- set interface vm3 type=internal
	ip link set vm3 address 02:ac:10:ff:01:94
	ip link set vm3 netns vm3
	ovs-vsctl set Interface vm3 external_ids:iface-id=inside-vm3
	ip netns exec vm3 dhclient vm3
	ip netns exec vm3 ip addr show vm3
	ip netns exec vm3 ip route show

On ubuntu3:

	ip netns add vm2
	ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
	ip link set vm2 address 02:ac:10:ff:01:31
	ip link set vm2 netns vm2
	ovs-vsctl set Interface vm2 external_ids:iface-id=dmz-vm2
	ip netns exec vm2 dhclient vm2
	ip netns exec vm2 ip addr show vm2
	ip netns exec vm2 ip route show
	
	ip netns add vm4
	ovs-vsctl add-port br-int vm4 -- set interface vm4 type=internal
	ip link set vm4 address 02:ac:10:ff:01:95
	ip link set vm4 netns vm4
	ovs-vsctl set Interface vm4 external_ids:iface-id=inside-vm4
	ip netns exec vm4 dhclient vm4
	ip netns exec vm4 ip addr show vm4
	ip netns exec vm4 ip route show



And testing connectivity from vm1 on ubuntu2:

	# ping the default gateway on tenant1
	root@ubuntu2:~# ip netns exec vm1 ping 172.16.255.129
	PING 172.16.255.129 (172.16.255.129) 56(84) bytes of data.
	64 bytes from 172.16.255.129: icmp_seq=1 ttl=254 time=0.689 ms
	64 bytes from 172.16.255.129: icmp_seq=2 ttl=254 time=0.393 ms
	64 bytes from 172.16.255.129: icmp_seq=3 ttl=254 time=0.483 ms

	# ping vm2 through the overlay
	root@ubuntu2:~# ip netns exec vm1  ping 172.16.255.131
	PING 172.16.255.131 (172.16.255.131) 56(84) bytes of data.
	64 bytes from 172.16.255.131: icmp_seq=1 ttl=64 time=2.16 ms
	64 bytes from 172.16.255.131: icmp_seq=2 ttl=64 time=0.573 ms
	64 bytes from 172.16.255.131: icmp_seq=3 ttl=64 time=0.446 ms

	# ping vm3 through the router, via the local ovs bridge
	root@ubuntu2:~# ip netns exec vm1  ping 172.16.255.194
	PING 172.16.255.194 (172.16.255.194) 56(84) bytes of data.
	64 bytes from 172.16.255.194: icmp_seq=1 ttl=63 time=1.37 ms
	64 bytes from 172.16.255.194: icmp_seq=2 ttl=63 time=0.077 ms
	64 bytes from 172.16.255.194: icmp_seq=3 ttl=63 time=0.076 ms
	
	# ping vm4 through the router, across the overlay
	root@ubuntu2:~# ip netns exec vm1  ping 172.16.255.195
	PING 172.16.255.195 (172.16.255.195) 56(84) bytes of data.
	64 bytes from 172.16.255.195: icmp_seq=1 ttl=63 time=1.79 ms
	64 bytes from 172.16.255.195: icmp_seq=2 ttl=63 time=0.605 ms
	64 bytes from 172.16.255.195: icmp_seq=3 ttl=63 time=0.503 ms


# Final Words
OVN makes layer-3 overlay networking relatively easy and pain free. The fact that services such as DHCP are built
directly into the system should help to reduce the number of external dependencies needed to build an effective
SDN solution. In the [next](http://blog.spinhirne.com/2016/09/the-ovn-gateway-router.html) post I will discuss ways of connecting our (currently isolated) overlay network to the outside world.




