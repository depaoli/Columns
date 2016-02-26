---
title: Quality Of Service
author: Matteo De Paoli
published: 2014-08-08
updated: 2014-08-08
tag: [bandwidth, cbwfq, cisco, congestion, dscp, lfi, llq, networking, phb, policing, pq, priority, qos, queue, red, shaping, tail-drop, wfq, wred]
headline: Overview of QoS, from TCP synchronization and fragmentation issues to IPSEC and the DSCP byte preservation feature, through LLQ, WRED, NBAR and other important technologies.
image: 
lead: This column is to give an overview of the QoS (Quality of Service) techniques that can be used on an IP network in order to provide "Congestion management" and to optimize "Congestion avoidance".

---

##Abstract

**QoS** is used to decide, when there is congestion, what traffic is critical (and must continue to work) and what traffic is not so critical (and may be drop).

>**NOTE**: Other than [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point), there are many technologies to obtain QoS, like [CoS](http://en.wikipedia.org/wiki/Class_of_service) for Ethernet and [CBR/VBR/etc.](http://en.wikipedia.org/wiki/Traffic_contract) in ATM networks.

**Congestion** happens when a router receive more traffic than what it can send (ie. when the bandwidth is not enough). This is a common situation on WAN interfaces, where the speed of the LAN interface/s is much higher than the WAN speed.  
When there is a congestion on an interface of a router, the packets are serialized in buffers called queues where they wait for their turn before going into the wire (causing an increasing of the latency). When the queues are full, the packets are discarded (usually, by default, with [Tail-Drop](http://en.wikipedia.org/wiki/Tail_drop)).

In TCP connections/[VC](http://en.wikipedia.org/wiki/Virtual_circuit#Examples_of_protocols_that_provide_virtual_circuits) (where [ACK](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Reliable_transmission) are used), when the queue grows and the latency increases, TCP notices the delay of the [ACK](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Reliable_transmission) it receives, and consequently it slows down the connection a bit.

![Effect of TCP Sync](http://1.bp.blogspot.com/-44hHGOq-r2I/U-YVMHCkfYI/AAAAAAAAKaA/L7Yf4fP8nTY/s1600/TCP+Sync.png){: style="float:right; width: 60%;"}

Then, if the queue become full and either [ACK](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Reliable_transmission) packets are not sent back from the remote host (because it doesn't receive discarded packets and consequently can't acknowledge) or [ACK](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Reliable_transmission) are discarded by a router on the return path, the peers of the TCP circuit (ie. end-hosts) use [TCP congestion control mechanism](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Congestion_control) to **lower their transmission speed much more**.

After that, TCP tries to increase the speed again until [ACK](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Reliable_transmission) will fail again (unless congestion ends).

Some mechanisms do not lower the bandwidth so much and try to keep the transmission rate just below the congestion level ([Congestion Avoidance](http://en.wikipedia.org/wiki/TCP_congestion_avoidance_algorithm), [Fast Retransmit](http://en.wikipedia.org/wiki/Fast_retransmit), etc.), but **when TCP believes that network is really congested, it drops the transmission rate so much** ([TCP Slow-start](http://en.wikipedia.org/wiki/Slow-start)).

When many TCP connections are in place in a congested network, it happens that multiple end-hosts drop their transmission rate at same time ([TCP Global Syncronization](http://en.wikipedia.org/wiki/TCP_global_synchronization)), then they raise up the speed again, in a kind of wave that **waste a lot of the bandwidth**.


##Latency management into the network level

To optimize the bandwidth utilization and to grant the proper bandwidth and latency to the applications we can act on different types of network delays:

- Fixed
    - Processing/Forwarding: the time used by router to process a packet and to apply things like encryption and compression ([enterprise class routers with proper capacity to process the required traffic in real time should be used](http://www.cisco.com/web/partners/downloads/765/tools/quickreference/routerperformance.pdf)).
    - **Serialization**: the time a router spends to put bits on the [media (ie. wire)](http://en.wikipedia.org/wiki/Transmission_medium), defined by interface speed and frame size.
    
    	<code>Serialization Delay (ms) = Frame size (Bytes) \*8 / bandwidth (kbps)</code>
    
	    **Huge frames mean higher latency at the same bandwidth**, because the single frame needs more time to be completely sent.
	          This delay do not impact the frame itself only, but if the Serialization delay is too long it could result in service degradation for the others high priority packets that must wait in the queue.
	     [Link Fragmentation and Interleaving (LFI)](http://docwiki.cisco.com/wiki/Quality_of_Service_Networking#Link_Efficiency_Mechanisms) could help if this is the case, by fragmenting packets to a smaller size and by interchange fragment of different packets (allowing QoS to prioritise traffic better). Anyway, the **further fragmentation means more overhead** (more headers for same payload), so it shouldn't be abused. In modern network we can avoid this, but it’s suggested where bandwidth is under 1Mbps.
	     ![Frame Serialization](http://3.bp.blogspot.com/-cwMYb5o0DGo/VXmLnzm-DWI/AAAAAAAALas/6Pk2fmZNqAc/s1600/frame%2Bserialization.jpg)
     
    - Propagation: the time required for the energy pulse (electrical/optical/waves/etc.) to traverse the [media](http://en.wikipedia.org/wiki/Transmission_medium) from source to destination, influenced mostly by the length of the link.

- Variable
    - **Queuing**: the time the packet stay in the router output queue waiting for its turn.


###PHB

[Per-Hop Behaviour](http://en.wikipedia.org/wiki/Per-hop_behaviour) is the individual **QoS action performed at each node of the network independently** (contrary to [IntServ](https://en.wikipedia.org/?title=Integrated_services), for instance, where QoS action is performed in synergy between all the nodes in a path).
Every QoS action influences the flow of the packets by enforcing some policies on the hops (ie. routers) that the packets are traversing, in an effort to obtain the desired quality for the services.

[PHB](http://en.wikipedia.org/wiki/Per-hop_behaviour) is usually obtained by defining the traffic flows through **Classification** and **Marking**, and by applying controlled **drops** to the traffic **queues**.


####Queuing and Dropping

When packets need to be sent out, they enter into a queue where they wait to be processed.
The **queuing** mechanism serialises the packets that go out of an interface. There are 2 different kinds of queues:

- Hardware Queues (tx/rx-ring): these queues end up on the [media](http://en.wikipedia.org/wiki/Transmission_medium), usually with a FIFO logic. Each interface can have only 1 hardware queue. They introduce delay, so they should NOT be very long. They also reduce the QoS power because hardware queue came after the software queues where QoS is applied (so, if it is too long, a prioritized packet might wait a lot of time in the hardware queue, regardless of QoS).
- Software Queues (Layer 3 queue): these queues are a kind of "programmable" queues, that allow a management of different types of flow in different ways:
    - FIFO: No QoS, First Input First Output.
    - PQ (Priority Queuing): By defining different priorities (eg. Hight/Medium/Normal/Low), the packets with higher priority will be sent before packets with lower priority. This can cause starvation to lower priority queues (because if high priority queue fill all the resources, the packets of lower priority queues will not be sent). Only the highest priority queue has delay and delivery granted.
    - Custom Queuing: It defines different classes, and for each of them an amount of data to store. When a specific queue is full, the new packets of its flow are discarded. The scheduler send out each queue sequentially. This means that greater is the queue size for a specific flow, greater will be the bandwidth assigned to it. Bandwidth can be guarantee but not delay (because a packet of the higher priority queue will wait the turn of its queue, regardless if it is the next one or the next 10th packet, and it could wait all the other 10 or more queues).
    - [WFQ (Weighted Fair Queuing)](http://en.wikipedia.org/wiki/Weighted_fair_queuing): It considers the packets as members of a common flow if their src/dst IP, src/dst ports and ToS values are equal. Each flow has its own queue (up to 4096). It doesn't guaranty delay nor bandwidth (it just ensure that each flow has its turn on the wire). It's "fair" in the meaning that one flow can't monopolize the bandwidth.
    - CBWFQ (Class Based Weighted Fair Queuing): It's like WFQ, but it allows to define classes of traffic and to assign them a different bandwidth. Each flow/packet will fall in these classes. It guarantees bandwidth but not delay.
    - [LLQ (Low-Latency Queuing)](http://en.wikipedia.org/wiki/Low-latency_queuing): Add PQ queue to CBWFQ. Every PQ packet that arrives on this queue is sent out immediately (to the hardware queue / tx-ring), before any CBWFQ queue. Because of this, it's mandatory to set a policier (max bandwidth), otherwise it will starve all other queues (eg. if bandwidth is 4 - [x][x][x][x] - and your policy to 1, you would have [x][x][x][llq], [x][x][x][llq], ... , but if you do not policy you could have [llq][llq][llq][llq], and fill the bandwidth).

**Preventive dropping** can be applied to every "software" queue to optimize the average usage of the bandwidth. There are different mechanisms:

- [Tail-Drop](http://en.wikipedia.org/wiki/Tail_drop): the most basic mechanism. When an interface receive more packets than supported, it fill the buffer (hardware queue) and then it drops every packet it receives (causing [Slow-start](http://en.wikipedia.org/wiki/Slow-start) and [TCP Global Syncronization](http://en.wikipedia.org/wiki/TCP_global_synchronization)).
- RED (Random Early Detection): it drops random packets on the queue before filling the bandwidth, causing TCP sources (eg. web browser/socket on PC, etc.) to slow down the connection without going in [Slow-start](http://en.wikipedia.org/wiki/Slow-start) (hopefully).
    - minimum threshold: below this value, no one packet is dropped
    - maximum threshold: starting from this value, every packet is dropped
    - MPD (Mark Probability Denominator): this is the percentage of dropped packets when the queue is between maximum and minimum threshold
- WRED (Weighted Random Early Detection): it's liker RED, but there are different MPD profiles for different [DSCP](http://en.wikipedia.org/wiki/Differentiated_services) values (it drops more the packets with less priority).


####Classification and Marking

The **classification of the different kind of traffic** (the flows) is essential and can be done by routers on following basis:

- ACL: ACLs permit to identify IPs, ports, etc.
- [NBAR](http://en.wikipedia.org/wiki/Network_Based_Application_Recognition): it's a Cisco technology that understand the protocols that use non standard or dynamic ports.
- DSCP: by looking at [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point) value of the [ToS](http://en.wikipedia.org/wiki/Type_of_service) field of the IP packet.

>**NOTE**: Before [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point), [IP Precedence](http://en.wikipedia.org/wiki/IP_precedence) codification was used in the [ToS](http://en.wikipedia.org/wiki/Type_of_service) field of the [IPv4 header](http://en.wikipedia.org/wiki/IPv4_header). Often, the [ToS](http://en.wikipedia.org/wiki/Type_of_service) word is used to identify the [IP Precedence](http://en.wikipedia.org/wiki/IP_precedence) codification, and [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point) is now used to indicate the [ToS](http://en.wikipedia.org/wiki/Type_of_service) field of the [IPv4 header](http://en.wikipedia.org/wiki/IPv4_header).

When the packet is classified, we can have it **marked**/remarked with a proper [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point) value/tag, in order to permit a faster processing of the packets (classification can be processor-intensive) in the successive hops (**PHB Policing/Marking**):

- DSCP marking [QC][DP]:
    - EF (46): low latency, low loss, low jitter (to use with a proper shaping because it may starve all other queues)
    - AF
        - Queue Classes
            1. Better than BE
            2. Better preference than 1
            3. Better preference than 2
            4. Better preference than 3
        - Drop Precedence
            1. lowest drop
            2. medium drop (dropped often than 1)
            3. highest drop (dropped often than 2)
    - BE (dscp 0)

    ***E.G.*** *AF31 is better than AF21, and AF 31 is better than AF32.*


###Traffic conditioners

Traffic conditioners allow to enforce speed limits to different types of traffic:

- **Policing**: can be applied in both inbound and outbound direction and drops the packets that exceed the limit.
- **Shaping**: can be applied in outbound only and, rather than dropping the exceeding packets, it tries to delay packets transmission by keeping the packets in a buffer during pick times, and by sending them later when bandwidth is not fully used. Of course, this behaviour cause delays because of the post-transmission of cached packets.

>**NOTE**: Compared with **policer** that do not buffer data, **shaper** optimize congested circuit and it is applied on the egress of an interface (an input queue should never be congested).

Worth to note that, because bits are sent into the wire at a standard clock rate, depending by line type, we can’t effectively send packets at a lower rate.

To accomplish the result, **the speed rate is simulated by interlacing periods of transmission (at line rate) with periods of non transmission**.


####CIR, Bc, Tc, Be and Token Buckets

Because of the interlacing periods described just above, the [CIR (Committed Information Rate)](http://en.wikipedia.org/wiki/Committed_information_rate) of a line is an average speed over the period of a second:

- Bc (Committed Burst): represents the amount of bits (for shaping) or bytes (for policing) stored in the token bucket at a given amount of time (Cisco uses the metaphor of tokens).
- Tc (Time Interval): the frequency at which the tokens are stored into the bucket.
- CIR = Bc / Tc
- Be (Excess Burst Rate): allows to sent more data if we didn’t use the line for enough time to collect enough tokens.


####Scheduling Mechanisms

The **Scheduling** is the logic used to decides which queues to process and in what order. It selects the queue from which the packet to send will be pulled (for instance, by choosing to send 1 packet of a specific queue every 2 packets of a most important queue), that is how many resources (bandwidth) assign to each type of traffic that is going out an interface.
Packets in the PQ are immediately pushed in the hardware queue, or dropped if they exceed the policer rate.

![Scheduling](http://4.bp.blogspot.com/-O6dQMrl-y1w/VXmJRUYN9sI/AAAAAAAALac/kyLB9nlNw-U/s1600/scheduling.jpg){: .center-block style="width: 75%;"}


###Considerations about WAN circuits

It's worth to note that effectiveness of QoS settings is influenced a lot by the type of the circuit in use.
Even if proper QoS settings in the outgoing interface permit an optimal bandwidth usage and manage the outgoing traffic as we want (with indirect influence on the incoming traffic as well), when packets are out of our router and our controlled network, some parameters like the [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point) tag are considered and managed properly only on some kind of circuits (and if included in the agreement with the carrier).

MPLS and other private or pseudo-private circuits usually honor [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point) and have different [SLA](http://en.wikipedia.org/wiki/Service-level_agreement) for different buckets of traffic, buckets that grant things like bandwidth, delay, jitter, etc.

DIA circuits (internet access) don't provide this capability, even because a packet flows through different ISP and lands on different destinations.


###Cisco Configuration Example


###Classification

Example of classification performed with NBAR v2 and ACLs (through a class-map).

    class-map match-any CRITICAL_TRAFFIC_IN
     match protocol ssh    #NBAR v2 classification
     match protocol www    #NBAR v2 classification
     match protocol dns    #NBAR v2 classification
     match access-group name MY_SERVERS    #ACL classification
    
    ip access-list extended MY_SERVERS
     permit tcp any host 10.0.0.1 eq 8080
     permit tcp any host 10.0.0.2 eq 8080


###Marking and Ingress Policing

Define what priority (DSCP value) must be applied to each type of traffic (already classified by the class-maps) that is coming in from an interface (and we do police p2p traffic before entering the router).

    policy-map TRAFFIC_IN
     class CRITICAL_TRAFFIC_IN
      set ip dscp ef    #DSCP marking
     class LESS_CRITICAL_TRAFFIC_IN
      set ip dscp af41  #DSCP marking
    ...
     class P2P_IN
        police 56000 conform-action transmit  exceed-action drop
    #policier in ingress, default FIFO dropping, P2P traffic exceeding 5600bps is discarded in ingress (also when there isn't congestion)
    ...

>**NOTE**: The application of DSCP tag in ingress is particularly useful when encrypted tunnels are used in egress, because it permits to avoid to use [QoS Pre-Classify](http://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/WAN_and_MAN/V3PN_SRND/V3PN_SRND/v3p_plan.html#wp1039474) and to use just [Cisco DSCP/ToS byte preservation feature](http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/18667-crypto-qos.html#t4) (more on "Cisco IPSec and QoS" section).


###Shaping

Define how many resources (bandwidth and delay) to assign for each type of traffic (based on the class-maps) that is going out an interface (through a policy-map), based on LLQ and WRED.

    class-map match-all CRITICAL_TRAFFIC_OUT
     match ip dscp ef    #Match classification on DSCP value basis (applied during the marking in ingress)
    ...

    policy-map TRAFFIC_OUT  #Applied in egress to the WAN
     class CRITICAL_TRAFFIC_OUT
        priority percent 30
    #30% of priority, low latency, FIFO dropping, exceeding traffic allowed if no congestion, dropped otherwise"
     class LESS_CRITICAL_TRAFFIC_OUT
        priority percent 5
    #5% of priority, low latency, FIFO dropping, exceeding traffic allowed if no congestion, dropped otherwise"
     class ANOTHER_CLASS_OUT
        bandwidth percent 5
    #5% of bandwidth, WRED dropping, exceeding traffic allowed if no congestion, queued/allocated in remaining bandwidth otherwise"
     class class-default
        fair-queue
         random-detect dscp-based
    #no bandwidth guarantee, WRED dropping

    policy-map INTERFACE_OUT
     class class-default
      shape average 50000000
       service-policy TRAFFIC_OUT
    #configuring shaper at 50Mbps for CBWFQ queues inside "TRAFFIC_OUT"

>**NOTE**: In this case we wrap the TRAFFIC_OUT in another Policy-map called INTERFACE_OUT to avoid to let it use the default interface speed for percent-based calculations.

On Cisco routers, **LLQ is obtained by using "priority" and "bandwidth" commands**.

First one defines the PQ reservation while second one defines WFQ/CBWFQ reservation.

**When there is no congestion**, [both commands allow their classes to use more bandwidth than their reserved allocation](http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10100-priorityvsbw.html#whichtrafficclassescanuseexcessbandwidth), with "priority" classes obtaining the tx-ring (a.k.a. hardware queue) at first.

**When there is a congestion**, PQ are policed to their limit (the traffic of a "priority" class above the allocated bandwidth is discarded), while WFQ/CBFWQ are allowed to use unutilized bandwidth from other classes [in proportion to their configured bandwidth](http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10100-priorityvsbw.html#howisunusedbandwidthallocated), and even to buffer packets in the Layer 3 queue (just before entering the tx-ring) until they reach the "queue-limit" ([former "shape max-buffers"](http://www.cisco.com/c/en/us/products/collateral/ios-nx-os-software/quality-of-service-qos/white_paper_c11-481499.html)).

>**NOTE**: In LLQ, if the PQ are using more than their reserved bandwidth, and this hurts the reservations of the WFQ/CBWFQ because congestion happens, PQ are policed, solving the starvation issue of pure PQ queuing.


###Cisco IPSec and QoS

When VPN (IPSec) tunnel is used, the same [DSCP](http://en.wikipedia.org/wiki/Differentiated_services_code_point) value of the unencrypted packet can be applied to the encrypted packet with the **[Cisco DSCP/ToS byte preservation feature](http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/18667-crypto-qos.html#t4)**.
Another useful feature is **[QoS Pre-Classify](http://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/WAN_and_MAN/V3PN_SRND/V3PN_SRND/v3p_plan.html#wp1039474)**, that permits the router to apply QoS on already encrypted packets on the basis of source port, destination IP, etc (keep in mind that, contrarily to the [Cisco DSCP/ToS byte preservation feature](http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/18667-crypto-qos.html#t4), [QoS Pre-Classify](http://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/WAN_and_MAN/V3PN_SRND/V3PN_SRND/v3p_plan.html#wp1039474) can be used only from the router that is encrypting the packet and not by the other routers on the path to the destination).

Also, when "***IPSec Anti-Replay Check Failures***" is enabled (and it should be), keep in mind that QoS could trigger out of order packets as showed in point 3 of [this doc](http://www.cisco.com/c/en/us/support/docs/ip/internet-key-exchange-ike/116858-problem-replay-00.html#anc9) and as explained [here](http://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/WAN_and_MAN/V3PN_SRND/V3PN_SRND/v3p_plan.html#wp1036005) (especially if huge frames are used, as described in the "Latency management into the network level" section of this column).


###Other useful IOS Commands

    #sh policy-map interface $INTERFACE
This command shows policy-maps (Service-policy) applied to the interface along with relative class-maps.
It prints out also the counters of dropped packets for each queue, and the allocated resources (it replaces "show queue interface" command in [IOS XE](http://www.cisco.com/c/en/us/td/docs/ios/ios_xe/qos/configuration/guide/convert/qos_mqc_xe/legacy_qos_cli_deprecation_xe.html#wp1099284) and [IOS 15](http://www.cisco.com/c/en/us/td/docs/ios/qos/configuration/guide/15_0s/qos_15_0s_book/legacy_qos_cli_deprecation.html#wp1099284)).

    #sh class-map $NAME
Shows the classification parameters used to mark the specific class-map. 

