
#################
# common dev env

FROM ubuntu:16.04
MAINTAINER gauchard@laas.fr
LABEL version="noisrev"

ENV PATH /root/bin:$PATH
ENV LD_LIBRARY_PATH /root/lib:$LD_LIBRARY_PATH
ENV MANPATH /root/man:$MANPATH
WORKDIR /root

#ADD files/sources.list-16.04-laas /etc/apt/sources.list
CMD mkdir -p /etc/ssh /root/.ssh /root/bin
ADD files/ssh/user/* /root/.ssh/
ADD files/ssh/sshd/* /etc/ssh/
RUN chmod 400 /etc/ssh/*_key; chmod 444 /etc/ssh/*.pub; chmod 700 /root/.ssh
RUN apt-get update && apt-get -y install apt-utils openssh-server

# end common
#################
# application build

RUN apt-get -y install iputils-ping net-tools tcpdump vim iptables telnet netcat-openbsd strace

ADD build* sendrecv.c ./

RUN ./build-sendrecv

#################
# default command must be sshd running on foreground 

CMD mkdir -p /var/run/sshd; /usr/sbin/sshd -D
