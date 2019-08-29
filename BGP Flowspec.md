# BGP Flowspec
BGP Flowspec is an alternative and a more granular method to RTBH described in RFC5575 that can be used to mitigate a distributed denial-of-service (DDoS) attack. Match criteria allow network operators to define a particular flow with source, destination, L4 parameters and packet specifics such as length. These are sent in a BGP UPDATE message to BGP border routers within FLOW_SPEC_NLRI along with the action criteria.  
Actions are carried in the extended communities Path attribute. They can drop traffic matching the flow specification, redirect traffic to a particular VRF for further analysis or policy traffic at a defined rate. 
BGP Flowspec resembles access lists or firewall filters that provide matching criteria and traffic filtering actions. They are injected to BGP and propagated to BGP peers. This mechanism allows an upstream AS system to perform inbound filtering in their ingress routers of traffic that a downstream AS wishes to drop or policy.  

## FlowSpec Actions  
RFC 5575 has defined 4 minimum Actions that routes matching FlowSpec NRLI types can take. These actions are carried as BGP extended communities added to the FlowSpec route. These actions are:
- Traffic-Rate Community
The Traffic-Rate community is non-transitive, that tells the receiving BGP peer, what to rate limit matching traffic to. If the traffic needs to be discarded or dropped, this will be limit of 0 should be used.
- Traffic-Action Community
The Traffic-Action community is used to sample defined traffic. This allows sampling and logging metrics to be collected from the FlowSpec route, that could be used to get a better understand of the attack traffic.
- Redirect Community
The Redirect community allows the FlowSpec traffic to be redirected into a Virtual Routing and Forward Instance VRF. As the same Route-Targets and Route-Distinguisher can be used, you are able to import routes into a dedicated blackhole VPN or any other VPNv4.
- Traffic-Marking Community
The Traffic-Marking community is used to modify the Differentiated Service Code Point DSCP bits of a transiting IP packet to the defined value. This could be used to set to FlowSpec routes to highest discard probability, allowing traffic not to dropped/discarded until co

Network infrastructure with the enabled BGP Flowspec consists of a Flowspec controller (server), one or more Flowspec clients. Rules that contain matching criteria and actions are created on the server and are redistributed to clients via MP-BGP. An additional optional route reflector component can be used to receive rules from the controller and distribute to its clients.

## Flow spec demonstration
<img width="638" alt="bgp flowspec" src="https://user-images.githubusercontent.com/50369643/63919523-969ef780-ca47-11e9-8627-9fa47632ddf2.png">

### Configuration of FlowSpec in the provider network

#### Edge router
<pre>
fisi@big# show protocols bgp group CUSTOM
local-as 64516;
neighbor 192.168.56.26 {
    family inet {
        unicast;
        <b>flow</b>;
    }
    peer-as 64512;
}
</pre>

#### Border router
<pre>
KH16#sh run | se router bgp
router bgp 64512
 bgp log-neighbor-changes
 neighbor CUSTOM peer-group
 neighbor CUSTOM remote-as 64516
 neighbor 192.168.56.36 peer-group CUSTOM
 !
 address-family ipv4
  neighbor 192.168.56.36 activate
 exit-address-family
 !
 <b>address-family ipv4 flowspec
  neighbor 192.168.56.36 activate</b>
 exit-address-family

<b>flowspec
 local-install interface-all</b>
 </pre>

### Verification

Border router
<pre>
KH16#sh bgp ipv4 unicast summary | b Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.56.36   4        64516      35      36        5    0    0 00:14:27        0

KH16#sh bgp ipv4 <b>flowspec</b> summary | b Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.56.36   4        64516      35      36        1    0    0 00:14:30        0
</pre>

Edge router
<pre>
fisi@big# run show bgp summary | find 192.168.56.26
192.168.56.26         64512         49         52       0       2       20:37 Establ
  inet.0: 0/0/0/0
  <b>inetflow.0: 0/0/0/0</b>
</pre>

## Configuration of Flowspec server to signal flowspec
Once the provider detects a DDoS attack targeting the servers 172.16.0.0/29. Service provider decides to drop all traffic to the prefix 172.16.0.0/29. To achieve this service provider will configure the flowspec server to tell other routers to drop traffic for the prefix 172.16.0.0/29

### Edge router

<pre>
fisi@big# show routing-options flow
route EXAMPLE {
    match destination 172.16.0.0/29;
    then discard;
}
term-order standard;
</pre>

## Verification

<pre>
fisi@big# run show firewall filter __flowspec_default_inet__

Filter: __flowspec_default_inet__
Counters:
Name                                                Bytes              Packets
172.16.0.0/29,*                                         0                    0
</pre>

Verify Flowspec routes because they are installed into their own routing table inetflow.0 and if dedicated, VRF for FlowSpec routes and the table will be under routing-instance-name.inetflow.0. 

<pre>
fisi@big# run show route table inetflow.0
inetflow.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
172.16.0.0/29,*/term:1
                   *[Flow/5] 00:06:41
                      Fictitious

fisi@big# run show route advertising-protocol bgp 192.168.56.26

inetflow.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  172.16.0.0/29,*/term:1
*                         Self                                    I
</pre>

The BGP update with Flowspec SAFI 133 is advertised to the provider border router .The UPDATE message contains flow specification, matching the 172.16.0.0/29 prefix, The action criterion that drops traffic matching flow specification, are carried as BGP extended communities. The border router programs the rules into TCAM. Traffic received from the internet that corresponds to the match criteria is dropped.

<pre>
KH16#sh bgp ipv4 flowspec summary | b Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.56.36   4        64516      35      36        1    0    0 00:14:30        <b>1</b>
</pre>

<pre>
KH16#sh bgp ipv4 flowspec
BGP table version is 1, local router ID is 10.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *    Dest:172.16.0.0/29
                      0.0.0.0                                0 64516 i
</pre>

<pre>
KH16#sh bgp ipv4 flowspec detail
BGP routing table entry for Dest:172.16.0.0/29, version 0
  Paths: (1 available, no best path)
  Not advertised to any peer
  Refresh Epoch 1
  64516, (FS invalid: originator)
    0.0.0.0 from 192.168.56.36 (1.1.1.1)
      Origin IGP, localpref 100, valid, external
      Extended Community: FLOWSPEC Traffic-rate:0,0
      rx pathid: 0, tx pathid: 0

KH16#show flowspec ipv4
AFI: IPv4
  Flow           :Dest:172.16.0.0/29
    Actions      :Traffic-rate: 0 bps  (bgp.1)

</pre>

When the attacker tries to access the servers again all traffic is dropped. We can test by simple ping

<pre>
attacker#do ping 172.16.0.1 re 10000
Type escape sequence to abort.
Sending 10000, 100-byte ICMP Echos to 172.16.0.1, timeout is 2 seconds:
................
Success rate is 0 percent (0/37)
</pre>
We can see all the 37 packets sent from the hacker to the server have been dropped by the border router in the provider network

<pre>
KH16#sh flowspec afi-all detail
AFI: IPv4
  Flow           :Dest:172.16.0.0/29
    Actions      :Traffic-rate: 0 bps  (bgp.1)
    Statistics                        (packets/bytes)
      Matched             :                  37/4218
      Dropped             :                  37/4218
</pre>

Note: In my testing I have observed traffic sourced from the border router or the edge router itself for example ping with source of loopback from the border router to the servers was working. Therefore the flowspec rule seem to be affecting only the transit traffic, will need more testing and reading the docs to verify this.

## Configuration of Flowspec Signalized from Customer to Provider  
Once Customer detects a DDoS attack targeting a destination UDP port 53 (DNS) on the servers 172.16.0.0/29, the BGP update is advertised to the provider edge router. The UPDATE message contains flow specification, matching the 172.16.0.0/29 prefix, UDP protocol (17) and destination port 53. Subsequently, the edge router sends the BGP update to the border router. Traffic received from the internet that corresponds to the match criteria is dropped. This time fow spec from the customer is more specific containing port and protocol to match this avoids blackholing all traffic. This could also be done in the option where the service provider flow spec server was signalling.

### Flowspec Server (Controller) Configuration on CE running IOSXR

Create class-map attack-fs matching the IP address of the servers, protocol (17) – UDP and destination port 53 (DNS).
<pre>
class-map type traffic match-all attack-fs
 match destination-address ipv4 172.16.0.0 255.255.255.248
 match protocol 17 
 match destination-port 53 
 end-class-map
</pre>

Configure policy-map attack_pbr and associate class-map attack-fs. Use the class-map to define an action.
<pre>
policy-map type pbr attack_pbr
 class type traffic attack-fs 
  drop
 class type traffic class-default 
 end-policy-map
</pre>

We need to define a route policy that allows exporting and importing routes to/from eBGP peers.
<pre>
route-policy PASS
  pass
end-policy 

router bgp 64501
 bgp router-id 10.4.4.4
 address-family ipv4 unicast
  network 172.16.0.0/24
 
 address-family ipv4 flowspec
  neighbor 10.0.0.2
  remote-as 64516
  address-family ipv4 unicast
   route-policy PASS in
   route-policy PASS out

address-family ipv4 flowspec
   route-policy PASS in
   route-policy PASS out
</pre>

Configure flowspec and assign service-policy type PBR policy-map attack_pbr.
<pre>
flowspec
 address-family ipv4
  service-policy type pbr attack_pbr
</pre>


## Verification on the CE router
<pre>
RP/0/0/CPU0:CE# show flowspec ipv4 detail
Sun Jul 28 12:07:41.925 UTC

AFI: IPv4
  Flow           :Dest:172.16.0.0/29,Proto:=17,DPort:=53
    Actions      :Traffic-rate: 0 bps (policy.1.attack_pbr.attack-fs)
RP/0/0/CPU0:CE#
</pre>
