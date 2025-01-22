
# A Primer on OVN
OVN is a [virtual networking](https://networkheresy.com/2015/01/13/ovn-bringing-native-virtual-networking-to-ovs/) platform
developed by the fine folks over at [openvswitch.org](http://openvswitch.org). The project was announced in early 2015
and has just recently released the first production ready version, version 2.6. In this posting I'll walk through the basics
of configuring a simple layer-2 overlay network between 3 hosts. But first, a brief overview of how the system functions.

OVN works on the premise of a distributed control plane where components are co-located on each node in the network.
The roles within OVN are:
* OVN Central -- Currently a single host supports this role and this host acts as a central point of API integration by
external resources such as a cloud management platform. The central control houses the OVN northbound database, which
keeps track of high-level logical constructs such as logical switches/ports, and the OVN southbound database which 
determines how to map logical constructs in ovn-northdb to the physical world.
* OVN Host -- This role is distributed amongst all nodes which contain virtual networking end points such as VMs. The 
OVN Host contains a "chassis controller" which connects upstream to the ovn-southdb as its authoratative source of 
record for physical network information, and southbound to OVS for which it acts as an openflow controller.



# The Lab 
My lab is running as a nested setup on a single esxi machine. The OVN lab itself will consist of 3 Ubuntu
16.04 servers connected to a common management subnet (10.127.0.0/25). The hosts and their IP addresses
are as follows:
* ubuntu1 10.127.0.2  -- will serve as OVN Central
* ubuntu2 10.127.0.3  -- will serve as an OVN Host
* ubuntu3 10.127.0.4  -- will serve as an OVN Host

The network setup for the lab is illustrated below.

![ovn lab](primer-lab.jpg "ovn lab")


For the sake of ease of testing I will simulate virtual machine workloads on the Ubuntu hosts by creating OVS 
internal interfaces and sandboxing them within a network namespace. The namespaces will ensure that our OVN overlay
network is completely isolated from the lab network.


# Building Open vSwitch 2.6
Open vSwitch version 2.6 was released on 2016/09/28 and may be downloaded from [here](http://openvswitch.org/releases/openvswitch-2.6.0.tar.gz). Since I am using a Ubuntu based system I found the instructions file INSTALL.Debian.md
included with the download to work well. Below is a brief summary of the instructions. You should build ovs on either one
of the three Ubuntu machines or on a dedicated build machine with an identical kernel version.

Update & install dependencies

	apt-get update
	apt-get -y install build-essential fakeroot

Install Build-Depends from debian/control file

	apt-get -y install graphviz autoconf automake bzip2 debhelper dh-autoreconf libssl-dev libtool openssl
	apt-get -y install procps python-all python-twisted-conch python-zopeinterface python-six

Check the working directory & build

	cd openvswitch-2.6.0
	
	# if everything is ok then this should return no output
	dpkg-checkbuilddeps
	
	`DEB_BUILD_OPTIONS='parallel=8 nocheck' fakeroot debian/rules binary`


The .deb files for ovs will be built and placed in the parent directory (ie. in ../). The next step is to 
build the kernel modules.

Install datapath sources

	cd ..
	apt-get -y install module-assistant
	dpkg -i openvswitch-datapath-source_2.6.0-1_all.deb 

Build kernel modules using module-assistant

	m-a prepare
	m-a build openvswitch-datapath
	
Copy the resulting deb package. Note that your version may differ slightly depending on your specific kernel version.

	cp /usr/src/openvswitch-datapath-module-*.deb ./


Transfer the following to all three Ubuntu hosts:
* openvswitch-datapath-module-*.deb
* openvswitch-common_2.6.0-1_amd64.deb
* openvswitch-switch_2.6.0-1_amd64.deb 
* ovn-common_2.6.0-1_amd64.deb
* ovn-central_2.6.0-1_amd64.deb
* ovn-host_2.6.0-1_amd64.deb


# Installing Open vSwitch

Install OVS/OVN + dependencies on ubuntu1:

	apt-get update
	apt-get -y install python-six python2.7
	dpkg -i openvswitch-datapath-module-*.deb
	dpkg -i openvswitch-common_2.6.0-1_amd64.deb openvswitch-switch_2.6.0-1_amd64.deb
	dpkg -i ovn-common_2.6.0-1_amd64.deb ovn-central_2.6.0-1_amd64.deb ovn-host_2.6.0-1_amd64.deb


Install OVS/OVN + dependencies on ubuntu2 and ubuntu3:

	apt-get update
	apt-get -y install python-six python2.7
	dpkg -i openvswitch-datapath-module-*.deb
	dpkg -i openvswitch-common_2.6.0-1_amd64.deb openvswitch-switch_2.6.0-1_amd64.deb
	dpkg -i ovn-common_2.6.0-1_amd64.deb ovn-host_2.6.0-1_amd64.deb


Once the packages are installed you should notice that ubuntu1 is now listening on the TCP ports associated
with  ovn-northd (6641) and the OVN southbound database (6642). Here is the output of a netstat on my system:

	root@ubuntu1:~# netstat -lntp
	Active Internet connections (only servers)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
	tcp        0      0 0.0.0.0:6641            0.0.0.0:*               LISTEN      1798/ovsdb-server
	tcp        0      0 0.0.0.0:6642            0.0.0.0:*               LISTEN      1806/ovsdb-server



If you do not see ovsdb-server listening on 6641/6642 then you may need to manually start the daemons via the init
scripts (/etc/init.d/openvswitch-switch and /etc/init.d/ovn-central).


# Creating the Integration Bridge
OVN will be responsible for managing a single bridge within OVS and anything that we wish to be connected to an
OVN logical switch must be connected to this bridge. Although we can name this bridge anything we want, the standard
convention is to name it "br-int". We'll want to verify that this bridge does not already exist and add it if needed.

On all ubuntu2/ubuntu3:

	ovs-vsctl list-br

If you do not see "br-int" listed in the output then you'll need to create the bridge manually. Remember, the
integration bridge must exist on all hosts which will house VMs.

	ovs-vsctl add-br br-int -- set Bridge br-int fail-mode=secure
	ovs-vsctl list-br

The "fail-mode=secure" is a security feature which configures the bridge to drop traffic by default. This is important
since we do not want tenants of the integration bridge to be able to communicate if our flows were somehow erased or
if we plugged tenants into the bridge before the OVN controller was started.


# Connecting the Chassis Controllers to the Central Controller
The next step in the setup is to connect the chassis controllers on ubuntu2/ubuntu3 to our central controller on
ubuntu1. 

On ubuntu2:

	ovs-vsctl set open . external-ids:ovn-remote=tcp:10.127.0.2:6642
	ovs-vsctl set open . external-ids:ovn-encap-type=geneve
	ovs-vsctl set open . external-ids:ovn-encap-ip=10.127.0.3


On ubuntu3:

	ovs-vsctl set open . external-ids:ovn-remote=tcp:10.127.0.2:6642
	ovs-vsctl set open . external-ids:ovn-encap-type=geneve
	ovs-vsctl set open . external-ids:ovn-encap-ip=10.127.0.4


These commands will trigger the OVN chassis controller on each host to open a connection to the OVN Central
host on ubuntu1. We've specified that our overlay networks should use the [geneve](https://tools.ietf.org/html/draft-gross-geneve-00)
protocol to encapsulate our data plane traffic.

You may verify the connectivity with netstat. The output on my ubuntu3 machine is as follows:

	root@ubuntu3:~# netstat -antp | grep 10.127.0.2
	tcp        0      0 10.127.0.4:39256        10.127.0.2:6642         ESTABLISHED 3072/ovn-controller



# Defining the Logical Network
Below is a diagram which illustrates the OVN logical components which will be created as part of this lab.

![ovn logical components](primer-logical.jpg "ovn logical components")

Lets start by define the OVN logical switch "ls1" along with its associated logical ports. From ubuntu1:

	# create the logical switch
	ovn-nbctl ls-add ls1

	# create logical port
	ovn-nbctl lsp-add ls1 ls1-vm1
	ovn-nbctl lsp-set-addresses ls1-vm1 02:ac:10:ff:00:11
	ovn-nbctl lsp-set-port-security ls1-vm1 02:ac:10:ff:00:11

	# create logical port
	ovn-nbctl lsp-add ls1 ls1-vm2
	ovn-nbctl lsp-set-addresses ls1-vm2 02:ac:10:ff:00:22
	ovn-nbctl lsp-set-port-security ls1-vm2 02:ac:10:ff:00:22

	ovn-nbctl show


In each command set we've created a uniquely named logical port. Normally these logical ports will be named with
a UUID to ensure uniqueness but for our purposes we'll use names which are more human friendly. We're also defining
which mac addresses we expect to be associated with each logical port and we take the further step of locking down
the logical port using port security (allows only the macs we've listed to source from the logical port).



# Adding "Fake" Virtual Machines
The next step is to create "virtual machines" which we'll connect to our logical switch. As mentioned before, we'll
use OVS internal ports and network namespaces to simulate virtual machines. We'll use the address space 172.16.255.0/24
for our logical switch.

On ubuntu2:

	ip netns add vm1
	ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
	ip link set vm1 netns vm1
	ip netns exec vm1 ip link set vm1 address 02:ac:10:ff:00:11
	ip netns exec vm1 ip addr add 172.16.255.11/24 dev vm1
	ip netns exec vm1 ip link set vm1 up
	ovs-vsctl set Interface vm1 external_ids:iface-id=ls1-vm1
	
	ip netns exec vm1 ip addr show


On ubuntu3:

	ip netns add vm2
	ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
	ip link set vm2 netns vm2
	ip netns exec vm2 ip link set vm2 address 02:ac:10:ff:00:22
	ip netns exec vm2 ip addr add 172.16.255.22/24 dev vm2
	ip netns exec vm2 ip link set vm2 up
	ovs-vsctl set Interface vm2 external_ids:iface-id=ls1-vm2
	
	ip netns exec vm2 ip addr show


If you've not worked much with namespaces or OVS then the above commands may be a bit confusing. In brief, we're
creating a network namespace named for our fake VM, adding an internal OVS port, adding the port to the namespace,
setting up the IP interface from within the namespace (netns exec commands), and finally creating an OVS interface
with an external_id set to the ID we defined for our logical port in OVN. Its this last bit which triggers OVS
to alert OVN that a logical port has just come online. OVN acts upon this notification by pushing instructions down
to the local chassis controller of the host machine which, in turn, pushes network flows down to OVS.

Note that I have explicitly set the mac addresses of these interfaces to match what we've defined in our OVN logical
switch configuration. This is important. The logical network will not work if the mac addresses are not properly mapped.
Keep in mind that I have actually reversed the workflow a bit here. Normally you would not change the mac address
of the VM, but rather push an existing, known mac into OVN/OVS. My goal was to make the mac/IP easily visible for 
demonstration purposes, thus I manually set them.


# Verifying and Testing Connectivity
From ubuntu1 we can verify the logical network configuration using ovn-sbctl. Here is the output from my system:

	root@ubuntu1:~# ovn-sbctl show
	Chassis "239f2c28-90ff-468f-a701-655585c630bf"
			hostname: "ubuntu3"
			Encap geneve
					ip: "10.127.0.4"
					options: {csum="true"}
			Port_Binding "ls1-vm2"
	Chassis "517d558e-158a-4cb2-8870-283e9d39685e"
			hostname: "ubuntu2"
			Encap geneve
					ip: "10.127.0.3"
					options: {csum="true"}
			Port_Binding "ls1-vm1"


Note the port bindings associated to each of our OVN Host machines. In order to test connectivity we'll simply
launch a ping from the namespace of vm1. Here is the output from my machine:

	root@ubuntu2:~# ip netns exec vm1 ping 172.16.255.22
	PING 172.16.255.22 (172.16.255.22) 56(84) bytes of data.
	64 bytes from 172.16.255.22: icmp_seq=1 ttl=64 time=1.60 ms
	64 bytes from 172.16.255.22: icmp_seq=2 ttl=64 time=0.638 ms
	64 bytes from 172.16.255.22: icmp_seq=3 ttl=64 time=0.344 ms



# Adding a 3rd "VM" and Migrating It
Lets add a 3rd "VM" to our setup and then simulate migrating it between hosts. First, define its logical port using 
ovn-nbctl on ubuntu1:

	ovn-nbctl lsp-add ls1 ls1-vm3
	ovn-nbctl lsp-set-addresses ls1-vm3 02:ac:10:ff:00:33
	ovn-nbctl lsp-set-port-security ls1-vm3 02:ac:10:ff:00:33

	ovn-nbctl show


Next, create the interface for this VM on ubuntu2.

	ip netns add vm3
	ovs-vsctl add-port br-int vm3 -- set interface vm3 type=internal
	ip link set vm3 netns vm3
	ip netns exec vm3 ip link set vm3 address 02:ac:10:ff:00:33
	ip netns exec vm3 ip addr add 172.16.255.33/24 dev vm3
	ip netns exec vm3 ip link set vm3 up
	ovs-vsctl set Interface vm3 external_ids:iface-id=ls1-vm3
	
	ip netns exec vm3 ip addr show


Test connectivity from vm3.

	root@ubuntu2:~# ip netns exec vm3 ping 172.16.255.22
	PING 172.16.255.22 (172.16.255.22) 56(84) bytes of data.
	64 bytes from 172.16.255.22: icmp_seq=1 ttl=64 time=2.04 ms
	64 bytes from 172.16.255.22: icmp_seq=2 ttl=64 time=0.337 ms
	64 bytes from 172.16.255.22: icmp_seq=3 ttl=64 time=0.536 ms


Note the OVN southbound DB configuration on ubuntu1. We see that ubuntu2 has 2 registered port bindings.

	root@ubuntu1:~# ovn-sbctl show
	Chassis "239f2c28-90ff-468f-a701-655585c630bf"
			hostname: "ubuntu3"
			Encap geneve
					ip: "10.127.0.4"
					options: {csum="true"}
			Port_Binding "ls1-vm2"
	Chassis "517d558e-158a-4cb2-8870-283e9d39685e"
			hostname: "ubuntu2"
			Encap geneve
					ip: "10.127.0.3"
					options: {csum="true"}
			Port_Binding "ls1-vm3"
			Port_Binding "ls1-vm1"


In order to simulate a migration of vm3 we'll delete the "vm3" namespace on ubuntu2, remove its port on br-int,
and then recreate the setup on ubuntu3. From ubuntu2:

	ip netns del vm3
	ovs-vsctl --if-exists --with-iface del-port br-int vm3
	ovs-vsctl list-ports br-int


From ubuntu3:

	ip netns add vm3
	ovs-vsctl add-port br-int vm3 -- set interface vm3 type=internal
	ip link set vm3 netns vm3
	ip netns exec vm3 ip link set vm3 address 02:ac:10:ff:00:33
	ip netns exec vm3 ip addr add 172.16.255.33/24 dev vm3
	ip netns exec vm3 ip link set vm3 up
	ovs-vsctl set Interface vm3 external_ids:iface-id=ls1-vm3


And test connectivity:

	root@ubuntu3:~# ip netns exec vm3 ping 172.16.255.11
	PING 172.16.255.11 (172.16.255.11) 56(84) bytes of data.
	64 bytes from 172.16.255.11: icmp_seq=1 ttl=64 time=1.44 ms
	64 bytes from 172.16.255.11: icmp_seq=2 ttl=64 time=0.407 ms
	64 bytes from 172.16.255.11: icmp_seq=3 ttl=64 time=0.395 ms


Again, note the OVN southbound DB configuration on ubuntu1. We see that the port binding has moved.

	root@ubuntu1:~# ovn-sbctl show
	Chassis "239f2c28-90ff-468f-a701-655585c630bf"
			hostname: "ubuntu3"
			Encap geneve
					ip: "10.127.0.4"
					options: {csum="true"}
			Port_Binding "ls1-vm2"
			Port_Binding "ls1-vm3"
	Chassis "517d558e-158a-4cb2-8870-283e9d39685e"
			hostname: "ubuntu2"
			Encap geneve
					ip: "10.127.0.3"
					options: {csum="true"}
			Port_Binding "ls1-vm1"


# Cleaning Up
Lets clean up the environment before quitting. 

From ubuntu1:

	# delete the logical switch and its ports
	ovn-nbctl ls-del ls1

From ubuntu2:

	# delete vm1
	ip netns del vm1
	ovs-vsctl --if-exists --with-iface del-port br-int vm1

From ubuntu3:

	# delete vm2 and vm3
	ip netns del vm2
	ovs-vsctl --if-exists --with-iface del-port br-int vm2

	ip netns del vm3
	ovs-vsctl --if-exists --with-iface del-port br-int vm3


# Final Words
As you can see, creating layer-2 overlay networks is relatively straight forward using OVN. In the 
[next](http://blog.spinhirne.com/2016/09/an-introduction-to-ovn-routing.html) post I will discuss
OVN layer-3 networking by introducing the OVN logical router.
