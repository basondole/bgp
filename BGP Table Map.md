# BGP Table Map  

This feature allows for selective download of prefixes from the BGP RIB to the routing table.
It is primary used to suppress the unnecessary downloading of certain BGP routes to the RIB or Forwarding Information Base (FIB) on a dedicated route reflector, which propagates BGP updates without carrying transit traffic.
Essesntially filtering the routes in RIB/FIB on the dedicated route-reflectors provides essential saving on the memory and CPU resources on the router.
This feature is applicable not only to route reflectors but also any router that require control of prefixes that are put in the routing table due to resources constraints


# Configurations
In below example the ASBR1 router receives 50 prefixes from its upstream however we are only interested in specific prefixes which we'll use the table map
<pre>
ASBR1#sh ver | i Software
Cisco IOS Software, 7200 Software (C7200-ADVIPSERVICESK9-M), Version 15.2(4)S5, RELEASE SOFTWARE (fc1)
BOOTLDR: 7200 Software (C7200-ADVIPSERVICESK9-M), Version 15.2(4)S5, RELEASE SOFTWARE (fc1)


ASBR1#sh bgp summary | b Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.29.0.2      4        64515       7       6      151    0    0 00:01:53       <b>50</b>

ASBR1#sh bgp nei 172.29.0.2 received-routes | count ^ \*
Number of lines which match regexp = <b>50</b>

ASBR1#sh ip route bgp | count ^B
Number of lines which match regexp = <b>50</b>
</pre>

Creating a prefix-list and route-map to match our intersting destinations
<pre>
ASBR1(config)#ip prefix-list PREFIXLIST_INTERSTING_DESTINATIONS seq 5 permit 10.1.0.0/16
ASBR1(config)#ip prefix-list PREFIXLIST_INTERSTING_DESTINATIONS seq 10 permit 192.168.21.0/24

ASBR1(config)#route-map MAP_BGPIB_TO_RIB permit 10
ASBR1(config-route-map)# match ip address prefix-list PREFIXLIST_INTERSTING_DESTINATIONS
ASBR1(config-route-map)#exit
</pre>

Applying the table-map
<pre>
ASBR1(config)#router bgp 64512
ASBR1(config-router)#address-family ipv4
ASBR1(config-router-af)#<b>table-map MAP_BGPIB_TO_RIB filter</b>
ASBR1(config-router-af)#exit
ASBR1(config-router)#exit
ASBR1(config)#do clear ip bgp 172.29.0.2 all
*Aug 11 22:54:12.887: %BGP-5-ADJCHANGE: neighbor 172.29.0.2 Down User reset
*Aug 11 22:54:12.887: %BGP_SESSION-5-ADJCHANGE: neighbor 172.29.0.2 IPv4 Unicast topology base removed from session  User reset
*Aug 11 22:54:13.823: %BGP-5-ADJCHANGE: neighbor 172.29.0.2 Up 
</pre>

## Verification
We are still receiving and accepting the 50 prefixes from the upstream but only 2 of those are allowed to the routing table
<pre>
ASBR1(config)#do sh bgp summary | b Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.29.0.2      4        64515       5       4      251    0    0 00:00:17       <b>50</b>
                                                     
ASBR1(config)#do sh bgp nei 172.29.0.2 received-routes | count ^ \*
Number of lines which match regexp = <b>50</b>

ASBR1(config)#do sh ip route bgp | count ^B
Number of lines which match regexp = <b>2</b>

ASBR1(config)#do sh ip route bgp | begin ^B
B        10.1.0.0 [20/0] via 172.29.0.2, 00:00:54
B     192.168.21.0/24 [20/0] via 172.29.0.2, 00:00:54

ASBR1(config)#do sh route-map
route-map MAP_BGPIB_TO_RIB, permit, sequence 10
  Match clauses:
    ip address prefix-lists: PREFIXLIST_INTERSTING_DESTINATIONS 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
ASBR1(config)#
ASBR1(config)#do sh ip prefix-list
ip prefix-list PREFIXLIST_INTERSTING_DESTINATIONS: 2 entries
   seq 5 permit 10.1.0.0/16
   seq 10 permit 192.168.21.0/24
ASBR1(config)#
</pre>
