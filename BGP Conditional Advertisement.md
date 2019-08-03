# BGP Conditional Advertisement
This features allows for advertisement of routes to other neighbours only when a specific condition is met. Particularly by checking the existence of another prefix in the BGP RIB such that the advertisement of a specific prefix or set of prefixes depends on the existence or inexistence of another prefix in the bgp table.

## Objective
AS64513 is the source of the prefix 172.31/16 which it announce to its two peers in AS64512 and AS64515 who in turn announce it to AS64516. AS64516 itself is a source of two prefixes 10.1.1/24 and 172.18.0/24. The objective is to have AS64516 announce the prefix 10.1.1/24 to AS64515 only when AS64515 announces the prefix 172.31/16 (from AS64513) to AS64516. When AS64515 is not advertising 172.31/16 to AS64516, AS64516 should stop advertising its prefix 10.1.1/24 to AS64515 while still advertising the other prefixes (except 10.1.1/24) to AS64515 as well as advertising 10.1.1/24 and other prefixes to AS64512.

<img width="674" alt="BGP Conditional advertisement" src="https://user-images.githubusercontent.com/50369643/62410542-86822c80-b5ef-11e9-97a0-df67ea571029.png">


## Configuration
### AS64516
<pre>
interface FastEthernet0/0
 description CONNECTION TO AS64515
 ip address 10.0.0.10 255.255.255.252
!
interface FastEthernet0/1
 description CONNECTION TO AS64512
 ip address 10.0.0.13 255.255.255.252
!
router bgp 64516
 neighbor 10.0.0.9 remote-as 64515
 neighbor 10.0.0.14 remote-as 64512
 !
 address-family ipv4
  network 10.1.1.0 mask 255.255.255.0
  network 172.18.0.0 mask 255.255.255.0
  neighbor 10.0.0.9 activate
  <b>neighbor 10.0.0.9 advertise-map MAP_AS64515_CONDITIONAL_OUT exist-map MAP_AS64513_FROM_AS64515</b>
  neighbor 10.0.0.9 soft-reconfiguration inbound
  neighbor 10.0.0.14 activate
  neighbor 10.0.0.14 soft-reconfiguration inbound
 exit-address-family
!
ip as-path access-list 1 permit 64515 64513.*
!
ip route 10.1.1.0 255.255.255.0 Null0
ip route 172.18.0.0 255.255.255.0 Null0
!
ip prefix-list PL_AS64513 seq 5 permit 172.31.0.0/16
!
ip prefix-list PL_AS64516 seq 5 permit 10.1.1.0/24
!
<b>route-map MAP_AS64513_FROM_AS64515 permit 10
 description MATCH AS64513 PREFIX RECEIVED FROM AS64515
 match ip address prefix-list PL_AS64513
 match as-path 1</p>
!
<b>route-map MAP_AS64515_CONDITIONAL_OUT permit 10
 match ip address prefix-list PL_AS64516</b>
</pre>

The route-maps applied tell the router to use ` MAP_AS64515_CONDITIONAL_OUT ` only when the conditions in ` MAP_AS64513_FROM_AS64515 ` are satisfied if the conditions are not satisfied it does the opposite of what is specified in ` MAP_AS64515_CONDITIONAL_OUT ` which in this case the opposite of ` MAP_AS64515_CONDITIONAL_OUT ` is to deny prefixes from prefix-list ` PL_AS64516` since the route map ` MAP_AS64515_CONDITIONAL_OUT ` does not have any other entries the implicit deny in the end of the route-map turns to allow all that is not matched with the prefix-list ` PL_AS64516`

The configuration on the other routers is just standard BGP stuff.

### AS64513
<pre>
<b>AS64513#sh run | se interface|route </b>
interface FastEthernet0/0
 description CONNECTION TO AS64515
 ip address 10.0.0.1 255.255.255.252
interface FastEthernet0/1
 description CONNECTION TO AS64512 
 ip address 10.0.0.5 255.255.255.252
router bgp 64513
 bgp log-neighbor-changes
 neighbor 10.0.0.2 remote-as 64512
 neighbor 10.0.0.6 remote-as 64515
 !
 address-family ipv4
  network 172.31.0.0
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.6 activate
 exit-address-family
ip route 172.31.0.0 255.255.0.0 Null0
</pre>

### AS64515
<pre>
<b>AS64515#sh run | se interface|route</b>
interface FastEthernet0/0
 description CONNECTION TO AS64516
 ip address 10.0.0.9 255.255.255.252
interface FastEthernet0/1
 description CONNECTION TO AS64513
 ip address 10.0.0.6 255.255.255.252
router bgp 64515
 neighbor 10.0.0.5 remote-as 64513
 neighbor 10.0.0.10 remote-as 64516
 !
 address-family ipv4
  neighbor 10.0.0.5 activate
  neighbor 10.0.0.10 activate
  neighbor 10.0.0.10 soft-reconfiguration inbound
  neighbor 10.0.0.10 route-map MAP_AS64516_OUT out
 exit-address-family
ip prefix-list PL_AS6513 seq 5 permit 172.31.0.0/16
route-map MAP_AS64516_OUT deny 10
 description DENY AS64513
 match ip address prefix-list PL_AS6513
route-map MAP_AS64516_OUT permit 20
 description PERMIT OTHERS
</pre>

### AS64512
<pre>
<b>AS64512#sh run | se interface|route</b>
interface FastEthernet0/0
 description CONNECTION TO AS64513
 ip address 10.0.0.2 255.255.255.252
interface FastEthernet0/1
 description CONNECTION TO AS64516 
 ip address 10.0.0.14 255.255.255.252
router bgp 64512
 bgp log-neighbor-changes
 neighbor 10.0.0.1 remote-as 64513
 neighbor 10.0.0.13 remote-as 64516
 !
 address-family ipv4
  network 192.168.0.0 mask 255.255.0.0
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.13 activate
  neighbor 10.0.0.13 soft-reconfiguration inbound
 exit-address-family
ip route 192.168.0.0 255.255.0.0 Null0
</pre>


