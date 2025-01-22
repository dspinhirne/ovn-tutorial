# The OVN Load Balancer
Building upon my [previous](20160927-The-OVN-Gateway-Router.md) post I 
will explore the the load balancing feature of OVN. But before getting started, lets review the setup
created in the last lab.

The lab network:

![ovn lab](primer-lab.jpg "ovn lab")

The OVN logical network:

![ovn logical components](primer-logical.jpg "ovn logical components")


# The OVN Load Balancer
The OVN load balancer is intended to provide very basic load balancing services to workloads
within the OVN logical network space. Due to its simple feature set it is not designed to replace
dedicated appliance-based load balancers which provide many more bells & whistles for
advanced used cases.

The load balancer uses a hash-based algorithm to balance requests for a VIP to an associated pool
of IP addresses within logical space. Since the hash is calculated using the headers of the client
request the balancing should appear fairly random, with each individual client request getting
stuck to a particular member of the load balancing pool for the duration of the connection. Load 
balancing in OVN may be applied to either a logical switch or a logical router. The choice of
where to apply the feature depends on your specific requirements. There are caveats to each approach.

When applied to a logical router, the following considerations need to be kept in mind:
1. Load balancing may only be applied to a "centralized" router (ie. a gateway router).
2. Due to point #1, load balancing on a router is a non-distributed service.

When applied to a logical switch, the following considerations need to be kept in mind:
1. Load balancing is "distributed" in that it is applied on potentially multiple OVS hosts.
2. Load balancing on a logical switch is evaluted only on traffic ingress from a VIF. This means
that it must be applied on the "client" logical switch rather than on the "server" logical switch.
3. Due to point #2, you may need to apply the load balancing to many logical switches depending on the
scale of your design.



# Using Our Fake "VMs" as Web Servers
In order to demonstrate the load balancer we want to create a pair of web servers in our "dmz" which will
serve up uniquely identifiable files. In order to keep things simple, we'll use a single line python 
web server running in our vm1/vm2 namespaces. Lets kick things off by starting up our web servers.

From ubuntu2:

	mkdir /tmp/www
	echo "i am vm1" > /tmp/www/index.html
	cd /tmp/www
	ip netns exec vm1 python -m SimpleHTTPServer 8000


From ubuntu3:

	mkdir /tmp/www
	echo "i am vm2" > /tmp/www/index.html
	cd /tmp/www
	ip netns exec vm2 python -m SimpleHTTPServer 8000


The above commands will create a web server, listening on TCP 8000, which will serve up a file that 
can be used to identify the vm which served the file.

We'll also want to be able to test connectivity to our web servers. For this we'll use curl from the
global namespace of our Ubuntu hosts. Be sure to install curl on them if its not already.

	apt-get -y install curl



# Configuring the Load Balancer Rules
As a first step we'll need to define our load balancing rules; namely the VIP and the back-end server
IP pool. All that is involved here is to create an entry in the OVN northbound DB and capture the
resulting UUID. For our testing we'll use the VIP 10.127.0.254 which resides within the "data" network
in the lab. We'll use the addresses of vm1/vm2 as our pool IPs.

From ubuntu1:

	uuid=`ovn-nbctl create load_balancer vips:10.127.0.254="172.16.255.130,172.16.255.131"`
	echo $uuid

The above command creates an entry in the load_balancer table of the northbound DB and stores the resulting
UUID to the variable "uuid". We'll reference this variable in later commands.




# The Gateway Router As a Load Balancer
Lets apply our load balancer profile to the OVN gateway router "edge1".

From ubuntu1:

	ovn-nbctl set logical_router edge1 load_balancer=$uuid

You can verify that this was applied by checking the database entry for edge1.

	ovn-nbctl get logical_router edge1 load_balancer


Now, from the global namespace of any of the Ubuntu hosts we can attempt to connect to the VIP.

	root@ubuntu1:~# curl 10.127.0.254:8000
	i am vm2
	root@ubuntu1:~# curl 10.127.0.254:8000
	i am vm1
	root@ubuntu1:~# curl 10.127.0.254:8000
	i am vm2

I ran the above tests several times and the load balancing appeared to be quite random.

Lets see what happens if we disable one of our web servers. Try stopping the python process running in the
vm1 namespace. Here is what I got:

	root@ubuntu1:~# curl 10.127.0.254:8000
	curl: (7) Failed to connect to 10.127.0.254 port 8000: Connection refused
	root@ubuntu1:~# curl 10.127.0.254:8000
	i am vm2

As you can see, the load balancer is not performing any sort of health checking. At present, the
assumption is that health checks would be performed by an orchestration solution such as Kubernetes
but it would be resonable to assume that this feature would be added at some future point.

Restart the python web server on vm1 before moving to the next test.

Load balancing works externally, but lets see what happens when we try to access the VIP from an
internal VM. Try using curl from vm3 on ubuntu2:

	root@ubuntu2:~# ip netns exec vm3 curl 10.127.0.254:8000
	i am vm1
	root@ubuntu2:~# ip netns exec vm3 curl 10.127.0.254:8000
	i am vm2

Nice. This seems to work as well, but also raises an interesting point. Take a second look at the logical
diagram for our OVN network and think about the traffic flow for the curl request from vm3. Also, look
at the logs from the python web server. Mine are below:

	10.127.0.130 - - [29/Sep/2016 09:53:44] "GET / HTTP/1.1" 200 -
	10.127.0.129 - - [29/Sep/2016 09:57:42] "GET / HTTP/1.1" 200 -

Notice the client IP addresses in the logs. The first is from ubuntu1 per the previous round of tests.
The second IP is edge1 itself and is from the request from vm3. Why is the request coming from edge1
rather than from vm3 directly? The answer is that the OVN developer who implemented load balancing
took into account something known as "proxy mode" where the load balancer hides the client side IP
under certain circumstances. Why is this necessary? Think about what would happen if the web server
saw the real IP of vm3. The response from the server would route back directly to vm3, bypassing the 
load balancer on edge1. From the perspective of vm3 it would look like it made a request to the VIP
but received a reply from the real IP of one of the web servers. This obviously would not work which
is why proxy-mode functionality is important.

Lets remove the load balancer profile and move on to a second round of tests.

	ovn-nbctl clear logical_router edge1 load_balancer
	ovn-nbctl destroy load_balancer $uuid



# Configuring the Load Balancer On a Logical Switch
Lets see what happens when we apply the load balancing rules to various logical switches within
our setup. Since we're moving load balancing away from the edge the first step we need is to 
create a new load balancer profile with an internal VIP. We'll use 172.16.255.62 for this.

From ubuntu1:
	
	uuid=`ovn-nbctl create load_balancer vips:172.16.255.62="172.16.255.130,172.16.255.131"`
	echo $uuid

As a first test lets apply it to the "inside" logical switch.

From ubuntu1:

	# apply and verify
	ovn-nbctl set logical_switch inside load_balancer=$uuid
	ovn-nbctl get logical_switch inside load_balancer

And test from vm3 (which resides on "inside"):

	root@ubuntu2:~# ip netns exec vm3 curl 172.16.255.62:8000
	i am vm1
	root@ubuntu2:~# ip netns exec vm3 curl 172.16.255.62:8000
	i am vm1
	root@ubuntu2:~# ip netns exec vm3 curl 172.16.255.62:8000
	i am vm2

This seems to work. Lets remove the load balancer from "inside" and apply it to "dmz".

From ubuntu1:

	ovn-nbctl clear logical_switch inside load_balancer
	ovn-nbctl set logical_switch dmz load_balancer=$uuid
	ovn-nbctl get logical_switch dmz load_balancer

And again test from vm3:

	root@ubuntu2:~# ip netns exec vm3 curl 172.16.255.62:8000
	^C

No good. It hangs. Lets try from vm1 (which resides on "dmz"):

	root@ubuntu2:~# ip netns exec vm1 curl 172.16.255.62:8000
	^C

Also no good. This highlights the requirement that load balancing be applied on the client's logical
switch rather than the server's logical switch.

Be sure to clean up. From ubuntu1:

	ovn-nbctl clear logical_switch dmz load_balancer
	ovn-nbctl destroy load_balancer $uuid


# Final Words
Basic load balancing is a very "nice to have" feature. Given that it is built directly into OVN means one less
piece of software to deploy within your SDN. While the feature set is minimal, I actually think that it
covers the needs of a very broad set of users. Given time, I also expect that certain limitations such as
lack of health checking will be addressed.

In my [next](http://blog.spinhirne.com/2016/10/ovn-and-acls.html) post I will look into network security using OVN.
