---
layout: post
title: "Passive monitoring of sockets on Linux"
description: "An introduction to INET_DIAG"
category: 
tags: []
---
{% include JB/setup %}

A project I have worked on lately required me to passively monitor the state of
a large number of TCP sockets. There are different ways of achieving this. One
can for example parse the content of `/proc/net/tcp`, or work some
tcpdump/libpcap-magic to get close to real-time information.

My only requirement was to collect a snapshot of the state of the sockets every
X second, so using libpcap and somehow track the each TCP connection would have
been overkill. Parsing text was also not very tempting, it is after all C we are
talking about, so I decided to use [Netlink](http://linux.die.net/man/7/netlink)
and the NETLINK_INET_DIAG socket-family. This is similar to what the convenient
[ss](http://linux.die.net/man/8/ss)-utility does. 

Netlink is used to transfer information between kernel and user-space, and
provides user-space applications with a socket-based interface.  Netlink allows
for easy access to several parts of the Linux-kernel and is one of my favourite
Linux-features. For example, the [Multi Network
Manager](https://github.com/kristrev/multi) uses Netlink and the
NETLINK_ROUTE-family to detect new network interfaces and configure the network
subsystem. A good introduction to Netlink and how the messages are build is
[this](http://www.linuxjournal.com/article/7356) LinuxJournal-article.

In order to better follow the rest of the post, you should have the source code
of my INET_DIAG example-application open. It can be found [here]
(https://github.com/kristrev/inet-diag-example/blob/master/inet_monitor.c).
Please note that I have chosen to do all the Netlink-message handling manually
to avoid adding dependencies to the example. Normally, you should use a library
like [libmnl](http://netfilter.org/projects/libmnl/) to handle Netlink-messages.

##Creating a socket

The first step for getting socket information is to create the Netlink-socket
and specify the correct netlink familiy. It can be done as follows:

    if((nl_sock = socket(AF_NETLINK, SOCK_DGRAM, NETLINK_INET_DIAG)) == -1){
        perror("socket: ");
        return EXIT_FAILURE;
    }


##Generating a request

After the socket is created, you request information about the sockets you are
interested in by creating a netlink message. The netlink message has to contain
a request-struct specifying information about the sockets you are interested in.
For internet sockets (TCP, UDP, ...), as used in the example, this struct is
called `inet_diag_req_v2`.

First, you specify which address family and protocol your are interested in
(IPv4 and TCP in the example):

    conn_req.sdiag_family = AF_INET;
    conn_req.sdiag_protocol = IPPROTO_TCP;

Second, you have to specify the states you are interested in. For TCP, the
states are defined in /include/net/tcp_states.h. I have requested to receive
sockets in every state but three:

    conn_req.idiag_states = TCPF_ALL & 
        ~(TCPF_SYN_RECV | TCPF_TIME_WAIT | TCPF_CLOSE);

Finally, in order to receive addition information, you have to set the bitmask
idiag_ext (for inet sockets). The possible values are listed in
linux/inet_diag.h. Here, I only want the additional socket information (which is
the
[tcp_info](http://lxr.free-electrons.com/source/include/uapi/linux/tcp.h#L148)-struct
for TCP sockets).

    conn_req.idiag_ext |= (1 << (INET_DIAG_INFO - 1));

After having filled out the required netlink-information (see source code), the
message is ready to be sent.

##Parsing a message

An INET_DIAG netlink message is split into multiple parts or messages, where
each message contains information about one socket. The example application
loops through the messages, only stopping when the last message (marked by type
set to  NLMSG_DONE) has been found,

Each INET_DIAG message starts with a
[inet_diag_msg-struct](http://lxr.free-electrons.com/source/include/uapi/linux/inet_diag.h),
which is followed by the INET_DIAG_MEMINFO-extension and then any extensions you
requested. The inet_diag_msg-struct contains generic information like source and
destination port and address. The example application outputs some of this
information, as well as some from the tcp_info-struct (contained in the
INET_DIAG_INFO-extension).

##Filtering

INET_DIAG provides a powerful filtering component that allows for quite advanced
matching. You can for example match ports between a range. The easiest way to
understand this is (unfortunately?) to look in the [source
code](http://lxr.free-electrons.com/source/net/ipv4/inet_diag.c#396).  However,
it is easier than it looks:

* A filter is a set of inet_diag_bc_op-structs. The code contains which
  comparison to be made, while yes/no is the offset for the inet_diag_bc_op
  containing the action to take depending on the outcome.
* To match for port, the desired port has to be stored in the no-field of the
  follow-up inet_diag_bc_op-struct. You have to use the _GE/_LE-codes.
* To match for IP/port, inet_diag_hostcond-struct has to be inserted after the
  inet_diag_bc_op-struct. You also have to use the \_COND-codes.
* To abort a comparison (i.e., you know the filter wont match), you have to make
  the len variable in `inet_diag_bc_run()` negative. What is contained in no has
  to be larger than the value of len.
* There are some restrictions to legal values of no. See `inet_diag_bc_audit()`.
* _S_ in the constant name means source, _D_ destination. 
* Values, like port, are stored in the structs in host order.

The example application contains one example of how to filter on ports.

##Useful links

* [sock_diag.c](http://lxr.free-electrons.com/source/net/core/sock_diag.c)
  - This is where all socket-types register their handlers.
* [inet_diag.c](http://lxr.free-electrons.com/source/net/ipv4/inet_diag.c)
  - This where the different inet-sockets register their handlers. Also, where
    filter is applied and the heavy-lifting for the generic information is done.
* [tcp_diag.c](http://lxr.free-electrons.com/source/net/ipv4/tcp_diag.c)
  - The TCP diag handler.
* [unix/diag.c](http://lxr.free-electrons.com/source/net/unix/diag.c) - Example
  of a non-inet handler, Unix sockets.
