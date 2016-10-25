
This is a docker-testbed where an OMPI bug related to network is highlighted.

* BY RUNNING THESE SCRIPTS, ANY EXISTING DOCKER INSTALLATION WILL BE REMOVED *
* THESE SCRIPTS SUPPOSE SYSTEMD IS INSTALLED AND RUNNING *

To prevent any harm, it is adviced to try this in a (non-docker) virtual machine
based on ubuntu-16.04.

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

* docker-stable is installed with the option "--userns-remap=default" so
  root in container cannot be root in host.

* two docker containers based on ubuntu-16.04 are run with ssh server and
  preinstalled keys to log in using only dedicated local keyfiles.

* two docker independant network bridges (which are regular linux bridges)
  are setup and for the purpose of this demo, and are able to communicate
  to each other
  (docker prevents this, but iptables commands in the postinstall
   script re-enable this. hence this script must be run also after a host
   reboot).

* instead of lauching mpirun in the host, the ./mpirun script ssh into one
  of the docker container so to start from the inside the real mpirun.

* the RUNME script starts docker containers then use the local mpirun script
  to start the application. But is does not stop the containers.
  so to restart the application, containers must be stopped first:
	./1-RUNME ompi -k

* to log into containers, instructions to do so are printed using:
	./1-RUNME ompi --debug
  then:
	./ssh-to-container 10.243.1.3 (or whatever IP is displayed)
