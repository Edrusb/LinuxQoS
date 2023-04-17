# LinuxQoS
Sample Network QoS using Linux kernel Features

## Description
this repository contains sample scripts to illustrate the powerful feature embedded into the Linux Kernel to setup Network QoS

## Licensing & Support
These scripts are provided as is and were found to match a particular need (mine), it may or may not match you need and is intended as illustration. You are free to use theses scripts, modify them, publish them according to the GPLv3 license. No support will be provided, read the doc (RTFM) take time to understand the underneath concepts (queues, filters, classes) as well as iptables related stuff

## Overall explanations
Each Linux Network Interface can receive a set of queues. Usually in Networking, QoS is applied on output of a device very rarely in input (because you don't decide the order of what you receive), The only case I've seen where QoS in input was interesting was for a device (a switch) that had not CPU power to handle all the packets addressed directly to it (at the opposit of packet to switch from one of its port to another), the QoS could be done in hardware this the switch was not consuming extra CPU cycle to implement the ingress QoS. Here with Linux, QoS is in software, thus take into account the possible slightly extra CPU cycle consumtion and apply it only in output.

By default in linux this is a FIFO queing (pfifo_fast). I replaced the default queue discipline from FIFO to SFQ (Stockastik faireness Queuing) only because it better balances the many sessions (TCP and UDP) that can land in the a such queue. You can create as many queue (tc qdisc) as you need per network interface (like eth0). These queues or better saying *queue disciplines* (**tc qdisc**) will hold packets from local applications to the network that *filters* (**tc filter**) has decided to move to a particular queue. In the same time the scheduler of each particular network interface (eth0 for example) will select a given queue according to a policy and tranmit a packet from that queue to the network. I chose the class based scheduling (**tc class**) because it is very flexible and allow one to do nested QoS (a global class limits the bandwidth of all exiting packet and you define different classes inside that in case of congestion get predifined reserved bandwidth fraction of the global class).

### Filters
you can base filters on a mark placed by netfilter (iptables), this is far the most flexible way of implementing filters. Note that without filter all packet go to the default queue, so there is no issue setting up qdisc and class before filters. 

> 
> iptables -t mangle -A PREROUTING  -i eth0 -d 1.2.3.0/24 -j MARK --set-mark 6
> # all packets sent to any IP of 1.2.3.0/24 subnet will be associated with the mark 6 
> # in the Linux kernel. The packet are not modified, this is not DSCP or CoS field.
> # Once the packet get emmited to the network the mark is forgotten.
> # Modifying DSCP field is done using iptables but that another topic 
>

A useful command to see how filter get applied is **tc filter dev eth0**

