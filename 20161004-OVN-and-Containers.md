# OVN and Containers
Following up on my [previous](20161003-OVN-and-ACLs.md) post, the subject
of this discussion is OVN integration with containers. By the end of this lab we will have created a
container host "VM" which houses a pair of containers. These containers will be tied direcly into an 
OVN logical switch and will be reachable directly from all VMs within the logical network.


# The OVN Container Networking Model
According to the man page for ovn-architecture, OVN's container networking strategy of choice is to use a 
VLAN trunk connection to the conainer host VM and require that traffic from each container be isolated within
a unique VLAN. This, of course, means that some level of coordination must take place between OVN and the
container host to ensure that they are in sync regarding which VLAN tag is being used for a given container.
It also places a certain level of responsibility on the container host to make sure that containers are properly
isolated from one another internally. 

Going into a bit more detail, the basic idea is that within OVN you will create a logical port representing
the connection to the host VM. You will then define logical ports for your containers, mapping them to the 
"parent" VM logical port, and defining a VLAN tag to use. OVN will then configure OVS flows which will map
VLAN tagged traffic from the parent VM's logical port to the appropriate container logical port. The diagram
below illustrates this design.

![ovn containers](ovn-containers.jpg "ovn containers")


# The Existing Setup
Take a moment to review the current setup prior to continuing.

The lab network:

![ovn lab](primer-lab.jpg "ovn lab")

The OVN logical network:

![ovn logical components](primer-logical.jpg "ovn logical components")



# Defining the Logical Network
For this lab we are going to create a new fake "VM", vm5, which will host our fake "containers". The new
VM will plug into the existing DMZ switch alongside vm1 and vm2. We're going to use DHCP for both the new
VM and its containers.

Before creating the logical port for vm5 we need to locate the DHCP options that we
created for the DMZ network during the [previous](http://blog.spinhirne.com/2016/09/an-introduction-to-ovn-routing.html)
lab. We'll query the OVN northbound DB directly for this information. Here is the output on my system.

	root@ubuntu1:~# ovn-nbctl list DHCP_Options
	_uuid               : 7e32cec4-957e-46fa-a4cc-34218e1e17c8
	cidr                : "172.16.255.192/26"
	external_ids        : {}
	options             : {lease_time="3600", router="172.16.255.193", server_id="172.16.255.193", server_mac="02:ac:10:ff:01:93"}

	_uuid               : c0c29381-c945-4507-922a-cb87f76c4581
	cidr                : "172.16.255.128/26"
	external_ids        : {}
	options             : {lease_time="3600", router="172.16.255.129", server_id="172.16.255.129", server_mac="02:ac:10:ff:01:29"}

We want the UUID for the "172.16.255.128/26" network (c0c29381-c945-4507-922a-cb87f76c4581 in my case). Capture this UUID
for use in later commands.

Lets create the logical port for vm5. This should look pretty familiar. Be sure to replace {uuid} with the UUID from the
DHCP options entry above. From ubuntu1:

	ovn-nbctl lsp-add dmz dmz-vm5
	ovn-nbctl lsp-set-addresses dmz-vm5 "02:ac:10:ff:01:32 172.16.255.132"
	ovn-nbctl lsp-set-port-security dmz-vm5 "02:ac:10:ff:01:32 172.16.255.132"
	ovn-nbctl lsp-set-dhcpv4-options dmz-vm5 {uuid}

Now we will create the logical ports for the containers which live on vm5. This process is nearly identical to
creating a normal logical port but with a couple of additional settings. From ubuntu1:

	# create the logical port for c51
	ovn-nbctl lsp-add dmz dmz-c51
	ovn-nbctl lsp-set-addresses dmz-c51 "02:ac:10:ff:01:33 172.16.255.133"
	ovn-nbctl lsp-set-port-security dmz-c51 "02:ac:10:ff:01:33 172.16.255.133"
	ovn-nbctl lsp-set-dhcpv4-options dmz-c51 {uuid}

	# set the parent logical port and vlan tag for c51
	ovn-nbctl set Logical_Switch_Port dmz-c51 parent_name=dmz-vm5
	ovn-nbctl set Logical_Switch_Port dmz-c51 tag=51

	# create the logical port for c52
	ovn-nbctl lsp-add dmz dmz-c52
	ovn-nbctl lsp-set-addresses dmz-c52 "02:ac:10:ff:01:34 172.16.255.134"
	ovn-nbctl lsp-set-port-security dmz-c52 "02:ac:10:ff:01:34 172.16.255.134"
	ovn-nbctl lsp-set-dhcpv4-options dmz-c52 {uuid}

	# set the parent logical port and vlan tag for c52
	ovn-nbctl set Logical_Switch_Port dmz-c52 parent_name=dmz-vm5
	ovn-nbctl set Logical_Switch_Port dmz-c52 tag=52

So the only real difference here is that we've set a parent_name and tag for the container logical ports. You
can validate these by looking at the database entries. For example, the output on my system:

	root@ubuntu1:~# ovn-nbctl find Logical_Switch_Port name="dmz-c51"
	_uuid               : ea604369-14a9-4e25-998f-ec99c2e7e47e
	addresses           : ["02:ac:10:ff:01:31 172.16.255.133"]
	dhcpv4_options      : c0c29381-c945-4507-922a-cb87f76c4581
	dhcpv6_options      : []
	dynamic_addresses   : []
	enabled             : []
	external_ids        : {}
	name                : "dmz-c51"
	options             : {}
	parent_name         : "dmz-vm5"
	port_security       : ["02:ac:10:ff:01:31 172.16.255.133"]
	tag                 : 51
	tag_request         : []
	type                : ""
	up                  : false


# Configuring vm5
The first thing to remember about this lab is that we're not using real VMs, but rather simulating them as ovs
internal ports directly on the Ubuntu hosts. For vm1-vm4 we've been creating these internal ports directly on br-int
but for vm5 our requirements are a bit different so we'll be using a dedicated ovs bridge. This bridge, called br-vm5,
will not be managed by OVN and will simulate how you might actually configure an ovs bridge internal to a real
container host VM. This bridge will provide local networking to both the VM and its containers and will be configured
to perform VLAN tagging. Below is a diagram illustrating how things will look when we're done.

![ovn container lab](ovn-container-lab.jpg "ovn container lab")


My lab setup is simple in that I'm placing the containers all on the same logical switch. However, there is no requirement
to do this since I can place the container logical port on any logical switch that I wish.

Lets get started. The first step is to create the setup for vm5. We're going to do this on the ubuntu2 host.
From ubuntu2:

	# create the bridge for vm5
	ovs-vsctl add-br br-vm5

	# create patch port on br-vm5 to br-int
	ovs-vsctl add-port br-vm5 brvm5-brint -- set Interface brvm5-brint type=patch options:peer=brint-brvm5

	# create patch port on br-int to br-vm5. set external id to dmz-vm5 since this is our connection to vm5 
	ovs-vsctl add-port br-int brint-brvm5 -- set Interface brint-brvm5 type=patch options:peer=brvm5-brint
	ovs-vsctl set Interface brint-brvm5 external_ids:iface-id=dmz-vm5

	# create vm5 within a namespace. vm5 traffic will be untagged
	ovs-vsctl add-port br-vm5 vm5 -- set interface vm5 type=internal
	ip link set vm5 address 02:ac:10:ff:01:32
	ip netns add vm5
	ip link set vm5 netns vm5
	ip netns exec vm5 dhclient vm5

Verify connectivity from vm5 by pinging its default gateway.

	root@ubuntu2:~# ip netns exec vm5 ping 172.16.255.129
	PING 172.16.255.129 (172.16.255.129) 56(84) bytes of data.
	64 bytes from 172.16.255.129: icmp_seq=1 ttl=254 time=0.797 ms
	64 bytes from 172.16.255.129: icmp_seq=2 ttl=254 time=0.509 ms
	64 bytes from 172.16.255.129: icmp_seq=3 ttl=254 time=0.404 ms



# Configuring the vm5 "Containers"
Now that vm5 is up and working we can configure its fake "containers". These are going to look almost exactly
like our fake "vms" with the exception that we're configuring them for vlan tagging.

	# create c51 within a namespace. c51 traffic will be tagged with vlan 51
	ip netns add c51
	ovs-vsctl add-port br-vm5 c51 tag=51 -- set interface c51 type=internal
	ip link set c51 address 02:ac:10:ff:01:33
	ip link set c51 netns c51
	ip netns exec vm5 dhclient c51

	# create c52 within a namespace. c52 traffic will be tagged with vlan 52
	ip netns add c52
	ovs-vsctl add-port br-vm5 c52 tag=52 -- set interface c52 type=internal
	ip link set c52 address 02:ac:10:ff:01:34
	ip link set c52 netns c52
	ip netns exec c52 dhclient c52

Verify connectivity.

	root@ubuntu2:~# ip netns exec c51 ping 172.16.255.129
	PING 172.16.255.129 (172.16.255.129) 56(84) bytes of data.
	64 bytes from 172.16.255.129: icmp_seq=1 ttl=254 time=1.33 ms
	64 bytes from 172.16.255.129: icmp_seq=2 ttl=254 time=0.420 ms
	64 bytes from 172.16.255.129: icmp_seq=3 ttl=254 time=0.371 ms

	root@ubuntu2:~# ip netns exec c52 ping 172.16.255.129
	PING 172.16.255.129 (172.16.255.129) 56(84) bytes of data.
	64 bytes from 172.16.255.129: icmp_seq=1 ttl=254 time=1.53 ms
	64 bytes from 172.16.255.129: icmp_seq=2 ttl=254 time=0.533 ms
	64 bytes from 172.16.255.129: icmp_seq=3 ttl=254 time=0.355 ms


# Final Words
As per the ovn-architecture guide, if you run containers directly on a hypervisor or otherwise patch them directly into
the integration bridge then they have the potential to completely bog down an OVN system depending on the scale of your
setup. The "nested" network solution is nice in that it greatly reduces the number of VIFs on the integration bridge
and thus minimizes the performance hit which containers would otherwise entail. Again, the point of this exercise was 
not to set up a real world container simulation, but rather to demonstrate the in-built container networking feature 
set of OVN.

