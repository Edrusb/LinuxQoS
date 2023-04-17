# LinuxQoS
Sample Network QoS using Linux kernel Features

## Description
this repository contains sample scripts to illustrate the powerful feature embedded into the Linux Kernel to setup Network QoS

## Licensing & Support
These scripts are provided as is and were found to match a particular need (mine), it may or may not match you need and is intended as illustration. You are free to use theses scripts, modify them, publish them according to the GPLv3 license. No support will be provided, read the doc (RTFM) take time to understand the underneath concepts (queues, filters, classes) as well as iptables related stuff

## Overall explanations
Each Linux Network Interface can receive a set of queues. Usually in Networking, QoS is applied on output of a device very rarely in input (because you don't decide the order of what you receive), The only case I've seen where QoS in input was interesting was for a device (a switch) that had not CPU power to handle all the packets addressed directly to it (at the opposit of packet to switch from one of its port to another), the QoS could be done in hardware this the switch was not consuming extra CPU cycle to implement the ingress QoS. Here with Linux, QoS is in software, thus take into account the possible slightly extra CPU cycle consumtion and apply it only in output.

By default in linux this is a FIFO queing (pfifo_fast). I replaced the default queue discipline from FIFO to SFQ (Stockastik faireness Queuing) only because it better balances the many sessions (TCP and UDP) that can land in the a such queue. You can create as many queue (tc qdisc) as you need per network interface (like eth0). These queues or better saying *queue disciplines* (**tc qdisc**) will hold packets from local applications to the network that *filters* (**tc filter**) has decided to move to a particular queue. In the same time the scheduler of each particular network interface (eth0 for example) will select a given queue according to a policy and tranmit a packet from that queue to the network. I chose the class based scheduling (**tc class**) because it is very flexible and allow one to do nested QoS (a global class limits the bandwidth of all exiting packet and you define different classes inside that in case of congestion get predifined reserved bandwidth fraction of the global class).

Let's see the process from packet emmited by an application up to its exist to the network:

### Filters
you can base filters on a mark placed by netfilter (iptables), this is far the most flexible way of implementing filters. Note that without filter all packet go to the default queue, so there is no issue setting up qdisc and class before filters. 

    iptables -t mangle -A PREROUTING  -i eth0 -d 1.2.3.0/24 -j MARK --set-mark 6
    # all packets sent to any IP of 1.2.3.0/24 subnet will be associated with the mark 6 
    # in the Linux kernel. The packet are not modified, this is not DSCP or CoS field.
    # Once the packet get emmited to the network the mark is forgotten.
    # Modifying DSCP field is done using iptables but that another topic 

Now that packet have been eventually marked we can setup a filter to define the queue to move packet to based on this mark:

    /sbin/tc filter add dev eth0 parent 1:0 protocol ip prio 2 handle 6 fw flowid 1:10
    # all packets associated with mark 6 will be placed in the queue which id is 1:10
    
A useful command to see how filter get applied is 

    tc filter dev eth0

### Queue Disciplines

we create one or more queue(s) to receive and hold packets pending for transmission:

     # getting back to the default qdisc (qdisc del)
    /sbin/tc qdisc del dev eth0 root
   
    # here we say global queue  "1:" for eth0 interface is of type "htb" and all non-filtered packets
    # will go to the subque 1:20 (defined later):
    /sbin/tc qdisc add dev eth0 root handle 1: htb default 20
   
    # under this master queue 1:1 four queues are create these
    /sbin/tc qdisc add dev eth0 parent 1:10 handle 10: sfq perturb 10
    /sbin/tc qdisc add dev eth0 parent 1:20 handle 20: sfq perturb 10
    /sbin/tc qdisc add dev eth0 parent 1:30 handle 30: sfq perturb 10
    /sbin/tc qdisc add dev eth0 parent 1:40 handle 40: sfq perturb 10

So in that example all paquets will go to the queue 1:20 except those sent to a node of the IP subnet 1.2.3.0/24 which go to queue 1:10 because they have been marked with value 6 that filter defined above places in queu 1:10

### Scheduler (class)
Now that queues get filled by packets, the scheduler's role is to decide which queue to take packets first. Here we uses classes (tc class) to have hierarchical classes:

    /sbin/tc class add dev $DEV classid 1:1 parent 1: htb rate "$MAX" ceil "$MAX"
    /sbin/tc class add dev $DEV classid 1:10 parent 1:1 htb rate "$R10_1" ceil "$MAX" prio 50
    /sbin/tc class add dev $DEV classid 1:20 parent 1:1 htb rate "$R10_4" ceil "$MAX" prio 50
    /sbin/tc class add dev $DEV classid 1:30 parent 1:1 htb rate "$R10_5" ceil "$MAX" prio 50
    /sbin/tc class add dev $DEV classid 1:40 parent 1:1 htb rate "$R2_1" ceil "$R2_1" prio 10

in the above example we setup a global class "1:1" which is limited (traffic shaping) at the value of "$MAX" bit/s the rate is the average rate allocated to the class in case of congestion (here not other class are on competition with class 1:1, so the rate is not very useful here).

Within class 1, four classes 1:10, 1:20, 1:30 and 1:40 are in competition. In case of congestion they will have the value provided by "rate" reserved to them but can use as much as the ceiling value "ceil" if no other class need available bandwidth. 

The link between the class and the qdisc is set in the qdisc definition seen above:

    /sbin/tc qdisc add dev eth0 parent 1:10 handle 10: sfq perturb 10
  
 this definition means that the queue "10:" defined here has the class 1:10 as parent. In other words, the packets taken in the contract (rate/ceil) of the class 1:10 are taken from queue 10: defined above.
 
In conclusion, and to end the packet emission process, when its the turn to class 1:10 to emit a packet regarding other classes definititons and parent class definition, the queue 10: is asked a packet that if present is sent over then network

## Troubleshooting and observations

A couple of very interesting command  to see the QoS in action is:

    tc -s class show dev eth0
    tc -s qdisc show dev eth0

## Putting things in the correct order
We followed above the order of transmission of a packet from the application to the network information card (NIC), now let's put back in place the objects regarding their definitions and dependencies in the Linux kernel:

    eth0 (the network interface)
    |
    |
    +---> class
    |       |
    |       |
    |       +---> qdisc
    |       |
    |       +---> filter
    |
    +---> iptables
    
In other words, to the eth0 interface is associated first a the root class "1:" which is associated child classes "1:10" to 1:20" which are in turn given qdisc "10:" to "40:". In parallel, filters define which packets got to a specific queue (if a packet match no filter it goes the default queue). Filter may depend on the packet itself (like the value of the DSCP) but also they can depend on iptables thanks to the FW MARK which is very convenient to setup compared to other possible filter types.

## Zoom out
we say sfq queues, there are other types
we say htb class to provide hierarchical QoS (aka nested QoS) it also provides priority queuing, there is alternative like pure priority queuing and so on
we have been user filter bases on FW MARK, there is many other way to setup filter

# Examples
the /examples subdirectory contains several scripts that were functionning at a time, you can use them as a base of 
 
