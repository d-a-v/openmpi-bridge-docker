
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

Two scripts are provided which are sufficient to trigger and highlight the bug.

	git clone https://github.com/d-a-v/openmpi-bridge-docker.git
	cd openmpi-bridge-docker
	./0-PREPAREME
	./1-RUNME ompi

To demonstrate that the docker installation is viable, the same example
generating the ompi bug is also runnable without bug using mpich:

	./1-RUNME mpich
	Hello world: processor 0 of 2
	i am 1, sending to 0
	i am 0, recv from 1
	Hello world: processor 1 of 2
	That is all for now!
	i am 1, sent done to 0

	./1-RUNME ompi
	i am 1, sending to 0
	Hello world: processor 0 of 2
	i am 0, recv from 1
	(frozen from here)

In more details:

* the mpi application is in img-test*/sendrecv.c

* docker-stable is installed

* two docker containers based on ubuntu-16.04 are run with ssh server and
  preinstalled keys to log in using only dedicated local keyfiles.

* two docker independant network bridges (which are regular linux bridges)
  are setup and for the purpose of this demo, and are able to communicate to
  each other (docker prevents this, but "iptables" commands re-enable this. 
  hence this script "zz-postinstall-docker-test" also called by
  "0-PREPAREME" - must be run also after a host reboot).

* in the final setup, these two bridges are not lying on the same host, and
  routing tables allow these local bridges to communicate to each other via
  routing tables.

* instead of lauching mpirun in the host, the ./mpirun script ssh into one
  of the docker container so to start from the inside the real mpirun.

* the RUNME script starts docker containers then use the local mpirun script
  to start the application. But is does not stop the containers.
  so to restart the application, the most simple is to stop containers first:

	./1-RUNME ompi -k

* to log into containers, instructions to do so are printed using:

	./1-RUNME ompi --debug
	
	[info]
	
	./ssh-to-container 10.243.1.3 (or whatever IP is displayed)

* using the --debug option, is is not necessary to stop the container to
  restart the application: simply pasting again the provided provided will
  do.
