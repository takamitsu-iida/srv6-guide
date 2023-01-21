# SRv6 L3VPN

PEルータにVRFを作ってL3VPNを構成します。VPNとしてエンドエンドの通信を届けますので、SRv6網内の経路はクリーンなままです。

<br><br>

## 構成

![構成](img/srv6_bgp.drawio.png)

<br>

## ping

CE05からPE04へのpingをキャプチャ。送信元アドレスと宛先アドレス、NextHeader、に注目。

![ping](img/srv6_l3vpn_ping.PNG)

<br>



## CR01が知っているSID

CR01はVRFを定義しませんので、自身の経路情報としては（格納先がないので）何も持っていません。

ですが、BGPのルートリフレクタ役を担っていますので、BGPのテーブルに全てのvpnv4の情報を保持しています。

```
RP/0/RP0/CPU0:CR01#show bgp vpnv4 unicast received-sids
Sat Jan 21 15:54:20.603 JST
BGP router identifier 1.1.1.1, local AS number 65000
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0x0
BGP main routing table version 65
BGP NSR Initial initsync version 1 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Next Hop                            Received Sid
Route Distinguisher: 1:1
*>i192.168.3.0/24     2001:db8:0:3::1                     2001:db8:0:3:42::
*>i192.168.4.0/24     2001:db8:0:4::1                     2001:db8:0:4:42::

Processed 2 prefixes, 2 paths
```

Route Distinguisher 1:1 の経路として２つあり、それぞれどのSIDが割り当てられているか、がわかります。

<br>

## PE03が採番したSID

`show segment-routing srv6 sid` で自身が割り当てたSIDの一覧がわかりますが、どの経路に、というところまではわかりません。

```
RP/0/RP0/CPU0:PE03#show segment-routing srv6 sid
Sat Jan 21 15:57:53.355 JST

*** Locator: 'a' ***

SID                         Behavior          Context                           Owner               State  RW
--------------------------  ----------------  --------------------------------  ------------------  -----  --
2001:db8:0:3:1::            End (PSP/USD)     'default':1                       sidmgr              InUse  Y
2001:db8:0:3:40::           End.X (PSP/USD)   [Gi0/0/0/0, Link-Local]           isis-core           InUse  Y
2001:db8:0:3:41::           End.X (PSP/USD)   [Gi0/0/0/1, Link-Local]           isis-core           InUse  Y
2001:db8:0:3:42::           End.DT4           'vrf1'                            bgp-65000           InUse  Y
2001:db8:0:3:43::           End.DT4           'default'                         bgp-65000           InUse  Y
```

経路とSIDの対応を知りたい場合は`show bgp vpnv4 unicast local-sids`を参照します。

```
RP/0/RP0/CPU0:PE03#show bgp vpnv4 unicast local-sids
Sat Jan 21 15:59:33.544 JST
BGP router identifier 1.1.1.3, local AS number 65000
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0x0
BGP main routing table version 138
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

Processed 2 prefixes, 3 paths
```

<br>

## PE03から見たVRF経路

PE03に作成したvrf1の経路を見てみます。

```
RP/0/RP0/CPU0:PE03#show route vrf vrf1
Sat Jan 21 16:00:51.297 JST

Codes: C - connected, S - static, R - RIP, B - BGP, (>) - Diversion path
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - ISIS, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, su - IS-IS summary null, * - candidate default
       U - per-user static route, o - ODR, L - local, G  - DAGR, l - LISP
       A - access/subscriber, a - Application route
       M - mobile route, r - RPL, t - Traffic Engineering, (!) - FRR Backup path

Gateway of last resort is not set

C    192.168.3.0/24 is directly connected, 02:51:54, GigabitEthernet0/0/0/2.10
L    192.168.3.1/32 is directly connected, 02:51:54, GigabitEthernet0/0/0/2.10
B    192.168.4.0/24 [200/0] via 2001:db8:0:4::1 (nexthop in vrf default), 02:17:27
```

iBGPで学習した経路 `B    192.168.4.0/24 [200/0] via 2001:db8:0:4::1 (nexthop in vrf default), 02:17:27` のnexthopはPE04のループバックになっています。
ですが、実際の通信に使うのはこのnexthopではなく、SIDの方です。経路情報からどのSIDを使うかを調べることもできます。

```
RP/0/RP0/CPU0:PE03#show route vrf vrf1 192.168.4.0 detail
Sat Jan 21 16:05:50.574 JST

Routing entry for 192.168.4.0/24
  Known via "bgp 65000", distance 200, metric 0
  Tag 65006, type internal
  Installed Jan 21 13:43:24.262 for 02:22:26
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
  Route version is 0x13 (19)
  No local label
  IP Precedence: Not Set
  QoS Group ID: Not Set
  Flow-tag: Not Set
  Fwd-class: Not Set
  Route Priority: RIB_PRIORITY_RECURSIVE (12) SVD Type RIB_SVD_TYPE_REMOTE
  Download Priority 3, Download Version 115
  No advertising protos.
```

<br>

## CR01の設定

関係する部分のみ。

```
!
router bgp 65000
 bgp router-id 1.1.1.1
 bgp cluster-id 1
 address-family ipv4 unicast
 !
 address-family vpnv4 unicast
 !
 neighbor 2001:db8:0:2::1
  remote-as 65000
  update-source Loopback0
  address-family vpnv4 unicast
  !
 !
 neighbor 2001:db8:0:3::1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
  address-family vpnv4 unicast
   route-reflector-client
  !
 !
 neighbor 2001:db8:0:4::1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
  address-family vpnv4 unicast
   route-reflector-client
  !
 !
!
```


インタフェースに設定するMTUは9000バイトにイーサネットヘッダ14バイトを加えて9014としています。
同じ意味合いでCSR1000vを設定するときにはMTUを9000バイトと設定します。

SRv6に関して特に留意すべきことはありません。
SRv6のロケータを定義して、ISIS設定に加えているだけです。

```
!
hostname CR01
clock timezone JST Asia/Tokyo
username root
 group root-lr
 group cisco-support
 secret 10 $6$gyKeD/jZuHdy2D/.$nrt1ShaKihimEdQr.ASYcpWsFzn3zHg4oNPxgC5CS05ppkFthJA2EklVviaeuhf5DNUVOWWvOuWPqLGPSpEbZ0
!
cdp
!
interface Loopback0
 ipv6 address 2001:db8:0:1::1/128
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
interface GigabitEthernet0/0/0/2
 cdp
 mtu 9014
 ipv6 enable
!
router isis core
 is-type level-2-only
 net 49.0000.0000.0000.0001.00
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
   metric 10
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv6 unicast
   metric 10
  !
 !
!
router bgp 65000
 bgp router-id 1.1.1.1
 bgp cluster-id 1
 address-family ipv4 unicast
 !
 address-family vpnv4 unicast
 !
 neighbor 2001:db8:0:2::1
  remote-as 65000
  update-source Loopback0
  address-family vpnv4 unicast
  !
 !
 neighbor 2001:db8:0:3::1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
  address-family vpnv4 unicast
   route-reflector-client
  !
 !
 neighbor 2001:db8:0:4::1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
  address-family vpnv4 unicast
   route-reflector-client
  !
 !
!
segment-routing
 !
 srv6
  logging locator status
  encapsulation
   source-address 2001:db8:0:1::1
  !
  locators
   locator a
    prefix 2001:db8:0:1::/64
   !
  !
 !
!
end
```

<br>

## PE03の設定

関連するところのみ。

eBGPでCE05と接続しますので、経路フィルタの設定が必須です。IOS-XRはeBGPに対してフィルタを設定しないと有効になりません。

BGPで学習したipv4 unicast経路にロケータを割り当てています。
VRFやトンネルを定義せずとも、これだけの設定でIPv4 over SRv6が動きます。

```
!
hostname PE03
clock timezone JST Asia/Tokyo
username root
 group root-lr
 group cisco-support
 secret 10 $6$HXiCBcfH1uoB....$pdeWBcXfZRYi8IIKFJvRL90dvOug/i91ox9Cc3Aji/nYbsg8jYXkDo9xJwcCunIDkDun14w5O1SAo2md2aeoX/
!
cdp
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
interface GigabitEthernet0/0/0/2
 ipv4 address 192.168.3.1 255.255.255.0
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
router bgp 65000
 bgp router-id 1.1.1.3
 address-family ipv4 unicast
  segment-routing srv6
   locator a
   alloc mode per-vrf
  !
 !
 address-family vpnv4 unicast
  vrf all
   segment-routing srv6
    locator a
   !
  !
 !
 neighbor 192.168.3.2
  remote-as 65005
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
 neighbor 2001:db8:0:1::1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   encapsulation-type srv6
   next-hop-self
  !
  address-family vpnv4 unicast
   next-hop-self
  !
 !
 neighbor 2001:db8:0:2::1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
   encapsulation-type srv6
   next-hop-self
  !
  address-family vpnv4 unicast
   next-hop-self
  !
 !
!
segment-routing
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
end
```

SRv6のヘッドエンド動作でカプセル化するソースアドレスは、この設定です。

```
!
segment-routing
 srv6
  encapsulation
   source-address 2001:db8:0:3::1
  !
```

BGPで学習したIPv4経路に対してSIDを付与するのは、この設定です。

```
!
router bgp 65000
 bgp router-id 1.1.1.3
 address-family ipv4 unicast
  segment-routing srv6
   locator a
   alloc mode per-vrf
  !
 !
```
