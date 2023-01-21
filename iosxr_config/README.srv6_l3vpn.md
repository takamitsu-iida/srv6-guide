# SRv6 フルレングスSID

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

<br>

## ISIS

ISISは自身のロケータの情報を配る。

End.XのSIDを作る。

- metric-style wide 必須
- point-to-point トラフィックエンジニアリングはp2pが前提
- distribute link-state トラフィックエンジニアリングで必須

```
RP/0/RP0/CPU0:PE03#show run router isis core
Fri Jan 20 17:19:11.644 JST
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

RP/0/RP0/CPU0:PE03#
```


## RPL


eBGPでの経路交換に必要。

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

vrf vrf1
 rd 1:1
 address-family ipv4 unicast
  import route-target
   1:1
  !
  export route-policy SET_COLOR_10
  export route-target
   1:1
  !
 !
!
```

経路に対してロケータを使い分けたい場合。

FlexAlgoで経路単位にアフィニティを分けたいなら、これが必要。

```
route-policy set_per_prefix_locator_rpl
  if destination in (10.1.1.0/24) then
    set srv6-alloc-mode per-vrf locator locator1
  elseif destination in (2.2.2.0/24) then
    set srv6-alloc-mode per-vrf locator locator2
  elseif destination in (3.3.3.0/24) then
    set srv6-alloc-mode per-vrf
  else
    drop
  endif
end-policy
!
router bgp 100
 vrf vrf_cust1
  address-family ipv6 unicast
   segment-routing srv6
    alloc mode route-policy set_per_prefix_locator_rpl
   !
  !
 !
!
```

<br>

## BGP

```
RP/0/RP0/CPU0:PE03#sh run router bgp 65000
Fri Jan 20 17:19:48.792 JST
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
 vrf uvrf
  address-family ipv4 unicast
   segment-routing srv6
    locator uloc_1
    alloc mode per-vrf
   !
   redistribute connected
  !
 !
 vrf vrf1
  address-family ipv4 unicast
   segment-routing srv6
    locator a
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

<br>

## Segment Routing

```
segment-routing
 !
 srv6
  logging locator status
  encapsulation
   source-address 2001:db8:0:3::1
  !
  locators
   locator a
    prefix 2001:db8:0:3::/64
   !
  !
 !
!
```

<br>

## show bgp vpnv4 unicast local-sids

vpnv4経路のsidを確認する。

`show bgp vpnv4 unicast received-sids` にすると受信したものだけを表示する。

```
RP/0/RP0/CPU0:PE03#show bgp vpnv4 unicast local-sids
Fri Jan 20 17:28:06.628 JST
BGP router identifier 1.1.1.3, local AS number 65000
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0x0
BGP main routing table version 81
BGP NSR Initial initsync version 1 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Local Sid                                   Alloc mode   Locator
Route Distinguisher: 1:1 (default for vrf vrf1)
*> 192.168.3.0/24     2001:db8:0:3:42::                           per-vrf      a
*>i192.168.4.0/24     NO SRv6 Sid                                 -            -
* i                   NO SRv6 Sid                                 -            -
*> 192.168.255.5/32   2001:db8:0:3:42::                           per-vrf      a
*>i192.168.255.6/32   NO SRv6 Sid                                 -            -
* i                   NO SRv6 Sid                                 -            -
Route Distinguisher: 1:2 (default for vrf uvrf)
*> 192.168.3.0/24     fd00:0:300:e000::                           per-vrf      uloc_1
*>i192.168.4.0/24     NO SRv6 Sid                                 -            -
* i                   NO SRv6 Sid                                 -            -

Processed 6 prefixes, 9 paths
```

<br>

## show bgp vrf ... local-sids

vrfごとに採番したsidを表示する。

```
RP/0/RP0/CPU0:PE03#show bgp vrf uvrf local-sids
Fri Jan 20 17:31:36.264 JST
BGP VRF uvrf, state: Active
BGP Route Distinguisher: 1:2
VRF ID: 0x60000002
BGP router identifier 1.1.1.3, local AS number 65000
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000002   RD version: 81
BGP main routing table version 81
BGP NSR Initial initsync version 1 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0

Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Local Sid                                   Alloc mode   Locator
Route Distinguisher: 1:2 (default for vrf uvrf)
*> 192.168.3.0/24     fd00:0:300:e000::                           per-vrf      uloc_1
*>i192.168.4.0/24     NO SRv6 Sid                                 -            -
* i                   NO SRv6 Sid                                 -            -

Processed 2 prefixes, 3 paths
RP/0/RP0/CPU0:PE03#
RP/0/RP0/CPU0:PE03#show bgp vrf vrf1 local-sids
Fri Jan 20 17:31:46.845 JST
BGP VRF vrf1, state: Active
BGP Route Distinguisher: 1:1
VRF ID: 0x60000001
BGP router identifier 1.1.1.3, local AS number 65000
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000001   RD version: 80
BGP main routing table version 81
BGP NSR Initial initsync version 1 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0

Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Local Sid                                   Alloc mode   Locator
Route Distinguisher: 1:1 (default for vrf vrf1)
*> 192.168.3.0/24     2001:db8:0:3:42::                           per-vrf      a
*>i192.168.4.0/24     NO SRv6 Sid                                 -            -
* i                   NO SRv6 Sid                                 -            -
*> 192.168.255.5/32   2001:db8:0:3:42::                           per-vrf      a
*>i192.168.255.6/32   NO SRv6 Sid                                 -            -
* i                   NO SRv6 Sid                                 -            -

Processed 4 prefixes, 6 paths
RP/0/RP0/CPU0:PE03#
```

<br>

## show route vrf uvrf 192.168.4.0 detail

vrfの経路を詳細に表示。SRv6のヘッドエンド動作に注目。

uSIDの場合。

```
RP/0/RP0/CPU0:PE03#show route vrf uvrf 192.168.4.0 detail
Fri Jan 20 17:34:32.593 JST

Routing entry for 192.168.4.0/24
  Known via "bgp 65000", distance 200, metric 0, type internal
  Installed Jan 20 17:08:13.751 for 00:26:18
  Routing Descriptor Blocks
    2001:db8:0:4::1, from 2001:db8:0:1::1
      Nexthop in Vrf: "default", Table: "default", IPv6 Unicast, Table Id: 0xe0800000
      Route metric is 0
      Label: None
      Tunnel ID: None
      Binding Label: None
      Extended communities count: 0
      Source RD attributes: 0x0000:1:2
      NHID:0x0(Ref:0)
      SRv6 Headend: H.Encaps.Red [f3216], SID-list {fd00:0:400:e002::}
  Route version is 0x1 (1)
  No local label
  IP Precedence: Not Set
  QoS Group ID: Not Set
  Flow-tag: Not Set
  Fwd-class: Not Set
  Route Priority: RIB_PRIORITY_RECURSIVE (12) SVD Type RIB_SVD_TYPE_REMOTE
  Download Priority 3, Download Version 7
  No advertising protos.
RP/0/RP0/CPU0:PE03#
```

フルレングスSIDの場合。

```
RP/0/RP0/CPU0:PE03#show route vrf vrf1 192.168.4.0 detail
Fri Jan 20 17:36:18.248 JST

Routing entry for 192.168.4.0/24
  Known via "bgp 65000", distance 200, metric 0
  Tag 65006, type internal
  Installed Jan 20 17:08:13.750 for 00:28:04
  Routing Descriptor Blocks
    2001:db8:0:4::1, from 2001:db8:0:1::1
      Nexthop in Vrf: "default", Table: "default", IPv6 Unicast, Table Id: 0xe0800000
      Route metric is 0
      Label: None
      Tunnel ID: None
      Binding Label: None
      Extended communities count: 0
      Source RD attributes: 0x0000:1:1
      NHID:0x0(Ref:0)
      SRv6 Headend: H.Encaps.Red [base], SID-list {2001:db8:0:4:42::}
  Route version is 0x15 (21)
  No local label
  IP Precedence: Not Set
  QoS Group ID: Not Set
  Flow-tag: Not Set
  Fwd-class: Not Set
  Route Priority: RIB_PRIORITY_RECURSIVE (12) SVD Type RIB_SVD_TYPE_REMOTE
  Download Priority 3, Download Version 65
  No advertising protos.
RP/0/RP0/CPU0:PE03#
```
