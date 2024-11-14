# Container ROH

Routing on host for cloud container machines


## Background

A scaling infrastructure is a common requirement for modern cloud networks. As machines are added, the infrastructure need to be adopted and new challenges may appear.

The main reason for creating this setup is the fact the single virtual lans do not scale infinitely. The more machines are added, the more overhead is present. This overhead is caused by BUM packets...broadcasts, unknown unicasts and multicast packets. Since the target cannot be determined due to the nature of the packet, these packets have to be send to all involved machines. With a growing infrastructure this may result in a significant traffic (up to a few percent). While some of these packets may be reduced (e.g. no using multicasts), some are mandatory for a working network (e.g. ARP requests). This problem cannot be solved, so the common approach is splitting up the network into multiple virtual lans or broadcast domains. An example is using one network per rack. This is easily scalable is a powerful infrastructure is connecting the racks.

Another requirement is failure tolerance. A failing cable or a failing switch may not result in an outtake. The usual solution is using stacked switches or other setups combinung two or more switches. Each machine in a rack is connected to two switches, using LACP or other bonding methods for implementing failover and load balancing. LACP on server side is easy and available for almost all operating systems. On the switch side the story is different. Almost all stackable switches support LACP, but the implementation is vendor specific. You cannot mix switches from different vendors in a LACP setup. In some cases even different models from the same vendor are not supported. So going the LACP way results in a vendor lock-in. This should be avoided.

## The solution: L3 networks

VLANs and LACP are working on layer 2 and depend on switch support. Technically the ASIC in the switch is using a lookup table (FIB) for sending packets to the right port(s); the table itself is populated by the switch control plane e.g. after MAC addresses are detected. Running all packet through the control plane is not feasible given the speeed of modern network connection.
Layer 3 is the next layer in the network stack, also refered to as routing layer. In this layer the MAC address of packets does not matter, only the IP address is used for forwarding. Similar to the FIB most modern "layer 3 switches" also implement a "RIB". It is a lookup table for IP addresses / subnets and their corresponding ports. Routing a packet instead of simple forwarding thus does not have a performance penalty. The table is again populated by the control plane, using static router configuration and especially dynamic routing protocols like BGP and OSPF.

## ROH - Routing on host

Given a network with dynamic routing, layer 2 network can be skipped completely. Each host has a target IP address that should be used for communication. The address itself is assigned to a virtual dummy or loopback interface using a host network range (e.g. /32 in IPv4). Physical interfaces are connected to switches supporting a routing protocol; a routing daemon on the host is announcing the target address as host route to the connected switches. With a correct setup this provides both failover (link failure detection) and loadbalancing. A failed link can be detected with heartbeats supported by the routing protocol, or faster detection methods e.g. based on BFD. Loadbalancing is supported natively by the linux kernel; ECMP (equal cost multi pathing) can distribute packets to multiple targets if their routing metrics are equal. A hashing steps decides which target is used for packet, similar to the hashing used in LACP setups. The same limitations are also present; you should select a good hashing algorithm to optimize target/link utilization.

## Unnumbered links

With the advent of IPv6 each interface / port is assigned with a link local address (LLA) by default. This address is built from the MAC address and is unique (in contrast to IPv4 LLA). So if an IPv6 enabled host interfaces is attached to a IPv6 enabled switch port, both sides have a valid and working IPv6 address. It can already be used for sending packets. This is the base for "unnumbered" links which allow sending packets to the remote side. BGP and other routing protocols can use this mechanism for establishing routing links. This document is focussing on BGP, similar functionality is also available in certain OSPF implementations.

## BGP unnumbered
In a BGP unnumbered setup the routing process is watching for link state changes on monitored interfaces. If an interface is getting up and a LLA is assigned, it can use a special IPv6 multicast address to get the IPv6 LLA of the remote side. It then tries to establish a BGP routing session to that address. Afterwards packets can start to flow according to routes exchanged with the remote side. If the BGP session is interrupted, the routing process will revoke routes learned by that session (failover).

## BGP extended next hop
So far this setup only involves IPv6. RFC 5549 and the newer 8950 defines how IPv4 routes can be announced to IPv6 peers. This is last building block for the ROH setup. The local target address can be a arbitrary IPv4 address, the connections to switches and the whole network infrastructure can be implemented with IPv6 LLAs only. This simplifies the switch configuration.

## Advantages of a ROH setup
Since routing allows for better scalability, the network can grow nearly infinitely. A typical spine/leaf network is only limited by the number of ports available at the spine switches; for larger setup another layer can be put on top (super-spine/spine/leaf) etc. 
Packets like ARP request do not flood the network anymore, since ARP is not used at all in this setup. All hosts are addressed by their target IP address only.

The requirements for switches in the setup are also simplified. It needs to support FIB and RIB (standard on all modern switches) and an implementation of the routing protocol selected. Other features like stacking, LACP etc. are not needed anymore. This also allows to use cheaper unlabeled switches with stripped down operating systems, e.g. Cumulus Linux.

## Drawbacks of ROH
While the switch configuration is easier in ROH, the host setup is more involved. It needs a routing daemon on each host. Part of this problem can be solved with the ansible roles in this repository.
The number of routes present on each host and each switch also significantly higher. But modern switches should be able to deal with 1000s of routes easily.
Debugging problems also becomes more complex, since more switches and processes are involved. A good monitoring is critical for easy observability.
The main limitation of ROH is the host lifecycle. Switch ports can either be used for routing, or using as switching ports. If all network interfaces are configured for ROH, no interface can be used for the initial deploy of the host. The configuration management either has to be able to change the port mode on switches (e.g. to routing mode after ROH is configured), or an extra interface has to be used for deployment. This will require extra network cables, switches etc, raising the effort needed.

The role in this repository assumes an extra interface is used for deployment and configuration.


# Implementation requirements

This repository contains an ansible role for the **host** setup of ROH. It assumes a current RedHat or Debian based distribution or derivate. FRr (https://frrouting.org/) is used as routing daemon. Please note that the current release 10 does not work correctly in the DHCP fallback setup, so the role is using release 9.

## DHCP fallback
Older switches might not support BGP unnumbered. As a fallback the role also supports a DHCP based configuration. See the DHCP configuration section for details

## Network connectivity delay
After starting the frr daemon, it might take some time until routing is established and network connections to other hosts are possible. To cope with this problem the role can setup a post start action that waits until external hosts are accessible. See configuration section for details.

# Configuration

The configuration is separated into common configuration for all hosts, and individual per host item.

## Global

| Item                   | Description                                                      | Default |
|---------------- ------ | ---------------------------------------------------------------- | ------- | 
| frr_version            | FRR release to install                                           | 9.1.1    |
| data_as                | BGP AS number to use for hosts, should be the same for all hosts | 65420    |
| uplink_peer_group_name | name of the FRR peer group for uplink switches                   | switches |
| fabric_mtu             | MTU to use for dummy and routing interfaces                      | 9000     |
| proxy                  | HTTP(S) proxy for installing the FRR package repository          | -none-   |
| bgp_keepalive          | BGP keepalive interval                                           | 1        |
| bgp_hold               | BGP hold values                                                  | 3        |
| bgp_server_password    | BGP password to authenticate to switch, **HAS TO BE GIVEN**      | -none-   |

## Per host

| Item            | Description                                    |
| --------------- | -----------------------------------------------|
| roh_interfaces  | name of the interface to be configured for ROH |
| data_ip_address | target IP address to announce with routing     |

## DHCP mode

| Item             | Description                             | Default |
| ---------------- | --------------------------------------- | ------- |
| use_dhcp         | use DHCP modes instead of unnumbered    | false   |
| legacy_bgp_peers | list of possible BGP peers to configure | []      |
|                  | (each containing fields "as" and "ip")  |         |

The legacy_bgp_peers setting may contain the address and BGP AS of all BGP peers present in an infrastructure. Unreachable peers (e.g. switches in a different rack) will be ignored by FRR.

## Network connectivity delay handling

| Item             | Description                                        | Default  |
| ---------------- | -------------------------------------------------- | -------- |
| roh_ping_timeout | Maximum number of seconds to wait for a ping reply | - none - |
| roh_ping_target  | IP address to send ICMP ping requests to           | - none - |

Depending on the network setup this can either be a global or host based setting. Both items have to be present; in this case ping is run as a post start command and tries to ping the given target host once. If no reploy is received within the timeout, ping will fail.


# Switch configuration
The actual switch configuration is beyond the scope of the repository. It depends on the vendor and model of the switches used, and might require additional items not covered here.

The following check list can be used as a starting point:

  * enable BGP
  * set an individual BGP AS per switch (can be private AS number)
  * use short keepalive timeouts or BFD for failure detection
  * create a BGP peer group with the "data_as" AS and the "bgp_server_password" as MD5 password
    (some switches might allow any external AS in peer group instead)
  * add all interface with unnumbered connections to the peer group
  * ensure that these interface are configured for routing, IPv6, the correct MTU etc.
  * you might need additional BGP peers for uplink connectivity

