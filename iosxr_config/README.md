# IOS-XRのSRv6設定例

- L3VPNは難しくない。
- EVPNはXRv9000では動作しないので検証できない。
- TEはちょっと難しい

https://content.cisco.com/chapter.sjs?uri=/searchable/chapter/content/en/us/td/docs/iosxr/ncs560/segment-routing/73x/b-segment-routing-cg-73x-ncs560/m-configuring-srv6-traffic-engineering-ncs5xx.html.xml#id_125396



<br><br>

## 基本構成


## インタフェース

```
!
interface Loopback0
 ipv6 address 2001:db8:0:3::1/128
!
interface MgmtEth0/RP0/CPU0/0
 shutdown
!
interface GigabitEthernet0/0/0/0
 cdp
 mtu 9014
 ipv6 enable
!
interface GigabitEthernet0/0/0/1
 cdp
 mtu 9014
 ipv6 enable
!
```

## ISIS

ISISは自身のロケータの情報を配る。End.XのSIDを作る。

- distribute link-state トラフィックエンジニアリングで必須
- metric-style wide 必須
- point-to-point トラフィックエンジニアリングはp2pが前提


```
!
router isis core
 net 49.0000.0000.0000.0003.00
 distribute link-state
 nsf ietf
 address-family ipv6 unicast
  metric-style wide
  router-id Loopback0
  segment-routing srv6
   locator a
   !
  !
 !
 interface Loopback0
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv6 unicast
   metric 10
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv6 unicast
   metric 20
  !
 !
!
```


## RPL

```
!
extcommunity-set opaque color_10
  10
end-set
!
route-policy PASS-ALL
  pass
end-policy
!
route-policy SET_COLOR_10
  set extcommunity color color_10
  pass
end-policy
!
```



## BGP

```
!
router bgp 65000
 bgp router-id 1.1.1.3
 address-family vpnv4 unicast
  vrf all
   segment-routing srv6
    locator a
   !
  !
 !
 neighbor 2001:db8:0:1::1
  remote-as 65000
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-self
  !
 !
 neighbor 2001:db8:0:2::1
  remote-as 65000
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-self
  !
 !
 vrf vrf1
  address-family ipv4 unicast
   segment-routing srv6
    alloc mode per-vrf
   !
  !
  neighbor 192.168.3.2
   remote-as 65005
   address-family ipv4 unicast
    route-policy PASS-ALL in
    route-policy PASS-ALL out
   !
  !
 !
!

```

## Segment Routing


```
!
segment-routing
 traffic-eng
  srv6
  !
  candidate-paths
   all
    source-address ipv6 2001:db8:0:3::1
   !
  !
  segment-lists
   srv6
   !
   segment-list seg1
    srv6
     index 1 sid 2001:db8:0:1:1::
     index 2 sid 2001:db8:0:2:1::
     index 3 sid 2001:db8:0:1:1::
    !
   !
   segment-list seg2
    srv6
     index 1 sid 2001:db8:0:1:1::
    !
   !
   segment-list seg3
    srv6
     index 1 sid 2001:db8:0:2:1::
     index 2 sid 2001:db8:0:1:1::
     index 3 sid 2001:db8:0:2:1::
    !
   !
  !
  policy p1
   srv6
   !
   color 10 end-point ipv6 2001:db8:0:4:42::
   candidate-paths
    preference 100
     explicit segment-list seg1
     !
    !
   !
  !
 !
 srv6
  locators
   locator a
    prefix 2001:db8:0:3::/64
   !
  !
 !
!
```