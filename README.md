
This is a docker-testbed where an OMPI bug related to network is highlighted.

It is today extremely useful to be able to run mpi in docker installations
inside HPCs for a number of reasons.  In this docker container setup, the
only network interface is seen as "eth0" and is bridged with another host
bridge interface. open-mpi has a problem with it.

Similar bugs have been reported and seem not to have been solved so far.
Here are some references:

https://svn.open-mpi.org/trac/ompi/ticket/3339

https://www.open-mpi.org/faq/?category=tcp#ip-virtual-ip-interfaces

https://github.com/open-mpi/ompi/issues/203

http://stackoverflow.com/questions/15227933/cluster-hangs-shows-error-while-executing-simple-mpi-program-in-c

http://users.open-mpi.narkive.com/nWyjoU60/ompi-users-problem-running-an-mpi-applicatio-n-on-nodes-with-more-than-one-interface

The below scripts build a docker scenario with two containers running a
parallel test with both openmpi and mpich.

* BY RUNNING THESE SCRIPTS, ANY EXISTING DOCKER INSTALLATION WILL BE REMOVED

To prevent any harm, it is adviced to try this in a (non-docker) virtual machine.

Two scripts are provided which are sufficient to trigger and highlight the
bug.  The running user needs not be root, but must be sudoer.

	$ git clone https://github.com/d-a-v/openmpi-bridge-docker.git
	$ cd openmpi-bridge-docker
	$ ./0-PREPAREME

To demonstrate that the docker installation is viable, the same example
generating the ompi bug is also runnable without bug using mpich:

	$ ./1-RUNME mpich
	Hello world: processor 0 of 2
	i am 1, sending to 0
	i am 0, recv from 1
	Hello world: processor 1 of 2
	That is all for now!
	i am 1, sent done to 0

	$ ./1-RUNME ompi
	i am 1, sending to 0
	Hello world: processor 0 of 2
	i am 0, recv from 1
	(frozenstuckhung from here)
	^C
	
	(stop openmpi containers, restart them, display/copy/paste the run/debug command:)
	$ ./1-RUNME ompi -k		# stop ompi containers
	$ ./1-RUNME ompi --debug	# restart containers but not the application, show application command
	$ ./ssh-to-container 10.243.1.3 " \
		mkdir -p '/tmp/'; \
		cd '/tmp/'; \
		mpirun  -mca plm_rsh_agent /home/user/openmpi-bridge-docker/ssh-to-container \
			--allow-run-as-root  -mca btl self,tcp \
			-mca btl_tcp_if_include eth0 \
			-mca oob_tcp_if_include eth0 \
			-mca btl_base_verbose 100 \
			-mca orte_debug 1 \
			-mca orte_debug_verbose 100 -mca orte_base_help_aggregate 0 \
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
	(frozenstuckhung here again, but port 1024 on 10.242.1.3 is accessible from everywhere
	 test from another terminal on host and on both containers:)

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
  "0-PREPAREME" must be run also after a host reboot).

* instead of starting mpirun from the host, the ./mpirun script ssh into one
  of the docker container so to start from the inside the real mpirun.

* the RUNME script starts docker containers then use the local mpirun script
  to start the application. But is does not stop the containers.
  To restart the application, the most simple is to stop containers first:

	./1-RUNME ompi -k

* to log into containers, instructions to do so are printed using:

	./1-RUNME ompi --debug
	
	[info]
	
	./ssh-to-container 10.242.1.3 # no args, container1

	./ssh-to-container 10.243.1.3 # no args, container2

* by using the --debug option, it is not necessary to stop the container to
  restart the application: simply pasting again the provided command will
  do.

* in the final setup, these two bridges are not lying on the same host. 
  Routing tables allow these local bridges to communicate to each other via
  routing tables.  Problems on the final setup leaded me to make this
  testbed heading to the same problem but able to run on a single host.
