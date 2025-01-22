# OVN and ACLs
Building upon my [previous](h20160929-The-OVN-Load-Balancer.md) post I will now
examine basic network security using OVN Access Control Lists. 


# OVN Access Control Lists and Address Sets
ACLs within OVN are implemented in the ACL table of the northbound DB and may be implemented using the
acl commands of ovn-nbctl. At present, ACLs may only be applied to logical switches but it is
conceivable that the ability to apply them to routers would be a future enhancement.

ACLs are applied and evaluated in one of two directions: ingress to a logical port from a workload (to-lport) 
and egress from a logical port to a workload (from-lport). Additionally, every ACL is assigned a
priority which determines their order of evaluation. The highest priority is evaluated first and ACLs may be
given identical priorities. However, in the case of two ACLs having an identical priority and both matching
a given packet, only one will be evaluated. Exactly which one ultimately matches is indeterminite 
meaning you can't really be sure  which rule will be applied in a given situation. Moral of the story: try
to use unique priorities in most cases.

The rules for matching in ACLs are based on the flow syntax from OVS and should look familiar to anyone with
a programming background. The syntax is explained in the "Logical_Flow table" section of the man page for
ovn-sb. Its worth a read. In particular you should pay attention to the section discussing the "!=" match rule.
It is also worth highlighting the point that you can not create an ACL matching on a port with type=router.

In order to reduce the number of entries in the ACL table you may make use of address sets which define
groups of identical type addresses. For example, a group of IPv4 addresses/networks, a group of mac
addresses, or a group of IPv6 addresses may be placed within a named address set. Address sets can then be
referenced by name (in the form of $name) within the match clause of an ACL.

Lets run through some samples.

	# allow all ip traffic from port "ls1-vm1" on switch "ls1" and allowing related connections back in
	ovn-nbctl acl-add ls1 from-lport 1000 "inport == \"ls1-vm1\" && ip" allow-related

	# allow ssh to ls1-vm1
	ovn-nbctl acl-add ls1 to-lport 999 "outport == \"ls1-vm1\" && tcp.dst == 22" allow-related

	# block all IPv4/IPv6 traffic to ls1-vm1
	ovn-nbctl acl-add ls1 to-lport 998 "outport == \"ls1-vm1\" && ip" drop


Note the use of "allow-related". This is doing exactly what it seems like in that, under the covers,
it is permitting related traffic through in the opposite direction (eg. responses, fragements, etc..).
In the second rule I have used allow-related in order to allow responses to ssh back from the server. 

Lets take a look at address sets. As mentioned earlier, address sets are groups of addresses of the
same type. Address sets are created using the database commands of ovn-nbctl and are applied to
ACLs using the name of the address set. Here are some examples:

	ovn-nbctl create Address_Set name=wwwServers addresses=172.16.1.2,172.16.1.3
	ovn-nbctl create Address_Set name=www6Servers addresses=\"fd00::1\",\"fd00::2\"
	ovn-nbctl create Address_Set name=macs addresses=\"02:00:00:00:00:01\",\"02:00:00:00:00:02\"

Note the use of double quotes with address sets containing the ":" character. You'll get an error
if you don't quote these.


# Lab Testing
Lets experiment with ACLs in our lab environment. Here is a quick review of the setup. 

The lab network:

![ovn lab](primer-lab.jpg "ovn lab")

The OVN logical network:

![ovn logical components](primer-logical.jpg "ovn logical components")



As a first step we'll open up external access to the servers in our DMZ tier by creating a static
NAT rule for each of them. 

From ubuntu1:

	# create snat-dnat rule for vm1 & apply to edge1
	ovn-nbctl -- --id=@nat create nat type="dnat_and_snat" logical_ip=172.16.255.130 \
	external_ip=10.127.0.250 -- add logical_router edge1 nat @nat
	
	# create snat-dnat rule for vm2 & apply to edge1
	ovn-nbctl -- --id=@nat create nat type="dnat_and_snat" logical_ip=172.16.255.131 \
	external_ip=10.127.0.251 -- add logical_router edge1 nat @nat

Test connectivity from ubuntu1.

	root@ubuntu1:~# ping 10.127.0.250
	PING 10.127.0.250 (10.127.0.250) 56(84) bytes of data.
	64 bytes from 10.127.0.250: icmp_seq=1 ttl=62 time=2.57 ms
	64 bytes from 10.127.0.250: icmp_seq=2 ttl=62 time=1.23 ms
	64 bytes from 10.127.0.250: icmp_seq=3 ttl=62 time=0.388 ms

	root@ubuntu1:~# ping 10.127.0.251
	PING 10.127.0.251 (10.127.0.251) 56(84) bytes of data.
	64 bytes from 10.127.0.251: icmp_seq=1 ttl=62 time=3.15 ms
	64 bytes from 10.127.0.251: icmp_seq=2 ttl=62 time=1.52 ms
	64 bytes from 10.127.0.251: icmp_seq=3 ttl=62 time=0.475 ms

Also, check that the VMs can connect to the outside world using the proper IPs.

	root@ubuntu2:~# ip netns exec vm1 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	64 bytes from 10.127.0.130: icmp_seq=1 ttl=62 time=3.05 ms

	root@ubuntu3:~# ip netns exec vm2 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	64 bytes from 10.127.0.130: icmp_seq=1 ttl=62 time=1.87 ms

	root@ubuntu1:~# tcpdump -i br-eth1
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on br-eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
	17:51:01.055258 IP 10.127.0.250 > 10.127.0.130: ICMP echo request, id 4565, seq 12, length 64
	17:51:01.055320 IP 10.127.0.130 > 10.127.0.250: ICMP echo reply, id 4565, seq 12, length 64
	
	17:51:56.378089 IP 10.127.0.251 > 10.127.0.130: ICMP echo request, id 4301, seq 6, length 64
	17:51:56.378160 IP 10.127.0.130 > 10.127.0.251: ICMP echo reply, id 4301, seq 6, length 64


Excellent. We can see from the tcpdump on ubuntu1 that the VMs are using their proper NAT addresses.
Lets apply some security policy. Firstly, we'll completely lock down the DMZ. 

	# default drop
	ovn-nbctl acl-add dmz to-lport 900 "outport == \"dmz-vm1\" && ip" drop
	ovn-nbctl acl-add dmz to-lport 900 "outport == \"dmz-vm2\" && ip" drop

Lets do a quick access check from ubuntu1.

	root@ubuntu1:~# ping 10.127.0.250
	PING 10.127.0.250 (10.127.0.250) 56(84) bytes of data.
	^C
	--- 10.127.0.250 ping statistics ---
	2 packets transmitted, 0 received, 100% packet loss, time 1007ms

	root@ubuntu1:~# ping 10.127.0.251
	PING 10.127.0.251 (10.127.0.251) 56(84) bytes of data.
	^C
	--- 10.127.0.251 ping statistics ---
	2 packets transmitted, 0 received, 100% packet loss, time 1007ms

DMZ servers are now unreachable externally, however we've also managed to kill their outbound
access.

	root@ubuntu2:~# ip netns exec vm1 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	^C
	--- 10.127.0.130 ping statistics ---
	2 packets transmitted, 0 received, 100% packet loss, time 1008ms

	root@ubuntu3:~# ip netns exec vm2 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	^C
	--- 10.127.0.130 ping statistics ---
	2 packets transmitted, 0 received, 100% packet loss, time 1008ms

Lets fix that.

	# allow all ip trafficand allowing related connections back in
	ovn-nbctl acl-add dmz from-lport 1000 "inport == \"dmz-vm1\" && ip" allow-related
	ovn-nbctl acl-add dmz from-lport 1000 "inport == \"dmz-vm2\" && ip" allow-related

And verify.

	root@ubuntu2:~# ip netns exec vm1 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	64 bytes from 10.127.0.130: icmp_seq=1 ttl=62 time=4.16 ms
	64 bytes from 10.127.0.130: icmp_seq=2 ttl=62 time=3.07 ms

	root@ubuntu3:~# ip netns exec vm2 ping 10.127.0.130
	PING 10.127.0.130 (10.127.0.130) 56(84) bytes of data.
	64 bytes from 10.127.0.130: icmp_seq=1 ttl=62 time=3.59 ms
	64 bytes from 10.127.0.130: icmp_seq=2 ttl=62 time=2.30 ms

Lets allow inbound https to the DMZ servers. 

	# allow tcp 443 in and related connections back out
	ovn-nbctl acl-add dmz to-lport 1000 "outport == \"dmz-vm1\" && tcp.dst == 443" allow-related
	ovn-nbctl acl-add dmz to-lport 1000 "outport == \"dmz-vm2\" && tcp.dst == 443" allow-related

Lets verify. For this we'll need something listening on tcp 443. I like to use ncat, so the first step
is to install it on all 3 Ubuntu hosts. It is actually part of the nmap package.

	apt-get -y install nmap

Now we can start a process to listen on 443. The process will terminate at the end of the connection but
you can use the -k flag to keep it going if you want.

From ubuntu2:

	ip netns exec vm1 ncat -l -p 443

From ubuntu3:

	ip netns exec vm2 ncat -l -p 443

And check connectivity from ubuntu1. If the connection succeeds then it will stay open until you terminate it.
If not, then it should time out after 1 second.

	root@ubuntu1:~# ncat -w 1 10.127.0.250 443
	^C

	root@ubuntu1:~# ncat -w 1 10.127.0.251 443
	^C

That worked. Lets secure the "inside" servers as well. We'll make it really tight, blocking
all outbound access and only allowing access to tcp 3306 from the dmz. We'll use an address set
for allowing access from the DMZ. Note the use of the single quotes in the "acl-add" commands for 
allowing access to 3306. This is important. We're actually referring to the address set by its literal name
prefixed with a '$' character. We don't want bash to interpret this as a variable which is why we use the single
quote.

	# create an address set for the dmz servers. they fall within a common /31
	ovn-nbctl create Address_Set name=dmz addresses=\"172.16.255.130/31\"

	# allow from dmz on 3306
	ovn-nbctl acl-add inside to-lport 1000 'outport == "inside-vm3" && ip4.src == $dmz && tcp.dst == 3306' allow-related
	ovn-nbctl acl-add inside to-lport 1000 'outport == "inside-vm4" && ip4.src == $dmz && tcp.dst == 3306' allow-related

	# default drop
	ovn-nbctl acl-add inside to-lport 900 "outport == \"inside-vm3\" && ip" drop
	ovn-nbctl acl-add inside to-lport 900 "outport == \"inside-vm4\" && ip" drop


Again, we'll use ncat to listen on our VMs but this time well start it on vm3/vm4

From ubuntu2:

	ip netns exec vm3 ncat -l -p 3306

From ubuntu3:

	ip netns exec vm4 ncat -l -p 3306


Check connectivity from dmz to inside:

	root@ubuntu2:~# ip netns exec vm1 ncat -w 1 172.16.255.195 3306
	^C

	root@ubuntu3:~# ip netns exec vm2 ncat -w 1 172.16.255.194 3306
		^C

That seems to have worked. One final check. Lets make sure that vm3/vm4 are isolated from each other.

	root@ubuntu2:~# ip netns exec vm3 ncat -w 1 172.16.255.195 3306
	Ncat: Connection timed out.

	root@ubuntu3:~# ip netns exec vm4 ncat -w 1 172.16.255.194 3306
	Ncat: Connection timed out.


# Clean up
Be sure to remove the ACLs and address sets prior to quitting. From ubuntu1:

	ovn-nbctl acl-del dmz
	ovn-nbctl acl-del inside
	ovn-nbctl destroy Address_Set dmz



# Final Words
Traditionally it has been that firewalling was performed on an in-path, layer-3 device
such as a router or dedicated firewall appliance. In this regard it may seem odd that OVN applies policy
on the logical switch, however in reality this approach is advantageous in that it creates an easy way to
secure workloads in an east-west fashion by enforcing security at the logical port level. This approach to
security has come to be known as "micro segmentation" due to the fact that it allows an administrator to
apply security policy in a very fine grained manner. When you think about it, much of the heirarchy that 
exists in traditional network design (think web-tier/db-tier) is due to the fact that network security could
previously only be done on some central appliance which sat between tiers. The micro segmentation approach
actually allows you to flatten your designs to the point where you may end up having a single logical 
switch for everything with the "tiers" describing the security policy rather than describing the network layout.

In the [next](http://blog.spinhirne.com/2016/10/ovn-and-containers.html) post I will cover the OVN model for 
enabling container networking.


