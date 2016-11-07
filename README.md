
This is a docker-testbed where an OMPI bug related to network is highlighted.

-----------------------------------------------------------

Update 7-nov-16: This bug is now solved thanks to https://github.com/open-mpi/ompi/issues/2320

The reason is because docker bridges are NATed, and talking from one bridge
to another is also NAT-ed.  Allowing to talk between bridges without NAT-ing
them do not transform source addresses so open-mpi is happy with recveived
packets.  An ongoing open-mpi patch will notice the user about such
incorrect packets.

	$ ./10-RUNME ompi
	i am 1, sending to 0
	Hello world: processor 0 of 2
	i am 0, recv from 1
	(frozenstuckhung from here)
	^C
	
	$ ./05-PREPAREME-FIXNAT

	$ ./10-RUNME  ompi 
	i am 1, sending to 0
	Hello world: processor 0 of 2
	i am 0, recv from 1
	i am 1, sent done to 0
	Hello world: processor 1 of 2
	That is all for now!

-----------------------------------------------------------

It is today extremely useful to be able to run mpi in docker installations
inside HPCs for a number of reasons.  In this docker container setup, the
only network interface is seen as "eth0" and is bridged with another host
bridge interface. open-mpi has a problem with it.

Similar bugs have been reported and seem not to have been solved so far.
Here are some references:

https://github.com/open-mpi/ompi/issues/160

https://svn.open-mpi.org/trac/ompi/ticket/3339

https://www.open-mpi.org/faq/?category=tcp#ip-virtual-ip-interfaces

https://github.com/open-mpi/ompi/issues/203

http://stackoverflow.com/questions/15227933/cluster-hangs-shows-error-while-executing-simple-mpi-program-in-c

http://users.open-mpi.narkive.com/nWyjoU60/ompi-users-problem-running-an-mpi-applicatio-n-on-nodes-with-more-than-one-interface

The below scripts build a docker scenario with two containers running a
parallel test with both openmpi and mpich.

* BY RUNNING THESE SCRIPTS, ANY EXISTING DOCKER INSTALLATION WILL BE REMOVED

To prevent any harm, it is adviced to try this in a non-dockered virtual
machine.  I did in qemu/kvm with ubuntu xenial, but I did it too on a real
host (without an installed docker) and it worked faster with no harm.

Two scripts are provided which are sufficient to trigger and highlight the
bug.  The running user needs not be root, but must be sudoer.

	$ git clone https://github.com/d-a-v/openmpi-bridge-docker.git
	$ cd openmpi-bridge-docker
	$ ./00-PREPAREME

To demonstrate that the docker installation is viable, the same example
generating the ompi bug is also runnable without bug using mpich:

	$ ./10-RUNME mpich
	Hello world: processor 0 of 2
	i am 1, sending to 0
	i am 0, recv from 1
	Hello world: processor 1 of 2
	That is all for now!
	i am 1, sent done to 0

	$ ./10-RUNME ompi
	i am 1, sending to 0
	Hello world: processor 0 of 2
	i am 0, recv from 1
	(frozenstuckhung from here)
	^C
	
	(stop openmpi containers)
	$ ./10-RUNME ompi -k

	(restart them, notice the displayed run/debug command:)
	$ ./10-RUNME ompi --debug
	
	(copy/paste the result from above command:)
	$ ./ssh-to-container 10.243.1.3 " \
		cd '/tmp/'; \
		mpirun -mca plm_rsh_agent /home/user/openmpi-bridge-docker/ssh-to-container \
			--allow-run-as-root \
			-mca btl self,tcp \
			-mca btl_tcp_if_include eth0 \
			-mca oob_tcp_if_include eth0 \
			-mca btl_base_verbose 100 \
			-mca orte_debug 1 \
			-mca orte_debug_verbose 100 \
			-mca orte_base_help_aggregate 0 \
			-host 10.242.1.3,10.243.1.3  'sendrecv' \
		"
	[... skip ...]
	[26284fbaacbf:00017] select: initializing btl component tcp
	[b4bf0f67d2ee:00024] select: initializing btl component tcp
	[26284fbaacbf:00017] select: init of component tcp returned success
	[26284fbaacbf:00017] select: initializing btl component self
	[26284fbaacbf:00017] select: init of component self returned success
	[b4bf0f67d2ee:00024] select: init of component tcp returned success
	[b4bf0f67d2ee:00024] select: initializing btl component self
	[b4bf0f67d2ee:00024] select: init of component self returned success
	[26284fbaacbf:00017] mca: bml: Using self btl to [[61052,1],0] on node 26284fbaacbf
	[b4bf0f67d2ee:00024] mca: bml: Using self btl to [[61052,1],1] on node b4bf0f67d2ee
	[b4bf0f67d2ee:00024] mca: bml: Using tcp btl to [[61052,1],0] on node 10.242.1.3
	[26284fbaacbf:00017] mca: bml: Using tcp btl to [[61052,1],1] on node b4bf0f67d2ee
	i am 1, sending to 0
	Hello world: processor 0 of 2
	[b4bf0f67d2ee:00024] btl: tcp: attempting to connect() to [[61052,1],0] address 10.242.1.3 on port 1024
	i am 0, recv from 1
	(frozenstuckhung here again, but port 1024 on 10.242.1.3 is accessible from everywhere,
	(here are tests from another terminal:)

	$ telnet 10.242.1.3 1024 # accessible from host
	Trying 10.242.1.3...
	Connected to 10.242.1.3.
	Escape character is '^]'.
	^D
	
	$ ./ssh-to-container 10.242.1.3 telnet 10.242.1.3 1024 # accessible from container1
	Warning: Permanently added '10.242.1.3' (ECDSA) to the list of known hosts.
	Trying 10.242.1.3...
	Connected to 10.242.1.3.
	Escape character is '^]'.
	^D
	
	$ ./ssh-to-container 10.243.1.3 telnet 10.242.1.3 1024 # accessible from container2
	Warning: Permanently added '10.243.1.3' (ECDSA) to the list of known hosts.
	Trying 10.242.1.3...
	Connected to 10.242.1.3.
	Escape character is '^]'.
	^D
	
		

In more details:

* the mpi application is in img-test*/sendrecv.c

* docker-stable is installed

* two docker containers based on ubuntu-16.04 are run with ssh server and
  preinstalled keys to log in using only dedicated local keyfiles.

* two docker independant network bridges (which are regular linux bridges)
  are setup for the purpose of this demo, and are able to communicate to
  each other (docker prevents this, but some iptables commands re-enable
  this.  Hence the script "zz-postinstall-docker-test" also called by
  "00-PREPAREME" must be run also after a host reboot).

* instead of starting mpirun from the host, the ./mpirun script ssh into one
  of the docker container so to start from the inside the real mpirun.

* the 10-RUNME script starts docker containers then use the local mpirun script
  to start the application. But is does not stop the containers.
  To restart the application, the most simple is to stop containers first:

	./10-RUNME ompi -k

* to log into containers, instructions to do so are printed using:

	./10-RUNME ompi --debug
	
	[info]
	
	./ssh-to-container 10.242.1.3 # no args, container1

	./ssh-to-container 10.243.1.3 # no args, container2

* by using the --debug option, it is not necessary to stop the container to
  restart the application: simply pasting again the provided command will
  do.

* the open-mpi version used is the one provided in ubuntu xenial (1.10.2). 
  However the same bug is triggered with the github openmpi version (as of
  master@16/10/31).  To work with it, change "if true" to "if false" on top of
  "img-testompi/build-ompi-10", run the 20-UNINSTALL script then restart the
  00-PREPAREME.  docker image build time will be longer.

* in the final setup, these two bridges are not lying on the same host. 
  Routing tables allow these local bridges to communicate to each other. 
  Problems on the final setup leaded me to make this testbed heading to the
  same problem but able to run on a single host.


Informations about network interfaces:

  inside a container:

  	$ ./10-RUNME ompi --debug
  	[...skip...]
  	$ ./ssh-to-container 10.243.1.2
	root@e2a265eb93fa:~# ifconfig -a
	eth0      Link encap:Ethernet  HWaddr 02:42:0a:f3:01:02  
	          inet addr:10.243.1.2  Bcast:0.0.0.0  Mask:255.255.255.0
	          inet6 addr: fe80::42:aff:fef3:102/64 Scope:Link
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:75 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:63 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:0 
	          RX bytes:9751 (9.7 KB)  TX bytes:8839 (8.8 KB)
	
	lo        Link encap:Local Loopback  
	          inet addr:127.0.0.1  Mask:255.0.0.0
	          inet6 addr: ::1/128 Scope:Host
	          UP LOOPBACK RUNNING  MTU:65536  Metric:1
	          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1 
	          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

	root@e2a265eb93fa:~# brtcl show
	(empty result)

  inside host:

  	brtestdocker1 and 2 are created by 00-PREPARE scripts, they are bridges.
  	docker0 is another bridge created by docker.
  	ens3 is the kvm interface.
  	eth0 above is bridged with brtestdocker2 on host.

  	$ ifconfig -a
	brtestdocker1 Link encap:Ethernet  HWaddr 02:42:b1:fb:3e:b7  
	          inet addr:10.242.1.1  Bcast:0.0.0.0  Mask:255.255.255.0
	          inet6 addr: fe80::42:b1ff:fefb:3eb7/64 Scope:Link
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:0 
	          RX bytes:536 (536.0 B)  TX bytes:648 (648.0 B)
	
	brtestdocker2 Link encap:Ethernet  HWaddr 02:42:07:c3:20:c4  
	          inet addr:10.243.1.1  Bcast:0.0.0.0  Mask:255.255.255.0
	          inet6 addr: fe80::42:7ff:fec3:20c4/64 Scope:Link
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:104 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:108 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:0 
	          RX bytes:12477 (12.4 KB)  TX bytes:11949 (11.9 KB)
	
	docker0   Link encap:Ethernet  HWaddr 02:42:70:66:b0:5e  
	          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
	          inet6 addr: fe80::42:70ff:fe66:b05e/64 Scope:Link
	          UP BROADCAST MULTICAST  MTU:1500  Metric:1
	          RX packets:65251 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:147747 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:0 
	          RX bytes:3958858 (3.9 MB)  TX bytes:862582169 (862.5 MB)
	
	ens3      Link encap:Ethernet  HWaddr 52:54:00:c4:71:f2  
	          inet addr:192.168.122.131  Bcast:192.168.122.255  Mask:255.255.255.0
	          inet6 addr: fe80::5054:ff:fec4:71f2/64 Scope:Link
	          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
	          RX packets:489753 errors:0 dropped:8 overruns:0 frame:0
	          TX packets:108040 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1000 
	          RX bytes:1036793469 (1.0 GB)  TX bytes:10231810 (10.2 MB)
	
	lo        Link encap:Local Loopback  
	          inet addr:127.0.0.1  Mask:255.0.0.0
	          inet6 addr: ::1/128 Scope:Host
	          UP LOOPBACK RUNNING  MTU:65536  Metric:1
	          RX packets:160 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:160 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1 
	          RX bytes:11840 (11.8 KB)  TX bytes:11840 (11.8 KB)

	$ brctl show
	bridge name     bridge id               STP enabled     interfaces
	brtestdocker1   8000.0242b1fb3eb7       no              veth7863173
	brtestdocker2   8000.024207c320c4       no              vethb86ec05
	docker0         8000.02427066b05e       no

