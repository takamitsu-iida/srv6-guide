# SR-MPLS EVPN

4台のPEにEVIを作成して、全てのCEルータが同一セグメント上にくるように

VPNとしてエンドエンドの通信を届けますので、SRv6網内の経路はクリーンなままです。

<br><br>

## 構成

![構成](img/mpls_evpn.drawio.png)

<br>

## 完成形

先に完成形を示します。

CE111から他のCEにpingできることを確認します。

最初の1パケットが欠けるのはシスコ機器でよく見られる現象で、ARPが未解決な状態で通信を開始したためです（EVPNの問題ではありません）。

```
CE111#ping 10.0.0.112
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.112, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/2 ms

CE111#ping 10.0.0.113
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.113, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/3 ms

CE111#ping 10.0.0.114
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.114, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/2 ms
```

CE111からみると他のCEは同じLAN上にいますので、ARPテーブルにエントリができています。

```
CE111#show arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.111              -   aabb.cc00.0700  ARPA   Ethernet0/0
Internet  10.0.0.112              4   aabb.cc00.0800  ARPA   Ethernet0/0
Internet  10.0.0.113              4   aabb.cc00.0900  ARPA   Ethernet0/0
Internet  10.0.0.114              1   aabb.cc00.0a00  ARPA   Ethernet0/0
CE111#
```

PE11のGig1でキャプチャしたパケットです。MPLSのラベルが2段構成になっています。
一段目のラベルは出口のPEに向けてのパス、二段目はEVPNのEVIを示したものです。

![CE111からCE113向けのping](img/from_ce111_to_ce113_ping.PNG)

【参考】[pcapngファイル](img/from_ce111_to_ce113_ping.pcapng)


<br><br>

## 設計パラメータ

- 各機器のループバックアドレス

| 装置 |  アドレス          |
| ---- | ----------------- |
| CR1  | 192.168.255.1/32  |
| CR2  | 192.168.255.2/32  |
| PE11 | 192.168.255.11/32 |
| PE12 | 192.168.255.12/32 |
| PE13 | 192.168.255.13/32 |
| PE14 | 192.168.255.14/32 |

- 装置間リンクのアドレス

| リンク     |  アドレス        |
| --------- | ---------------- |
| CR1-PE11  | 192.168.111.0/24 |
| CR1-PE12  | 192.168.112.0/24 |
| CR1-PE13  | 192.168.113.1/24 |
| CR1-PE14  | 192.168.114.1/24 |
| CR2-PE11  | 192.168.211.0/24 |
| CR2-PE12  | 192.168.212.0/24 |
| CR2-PE13  | 192.168.213.1/24 |
| CR2-PE14  | 192.168.214.1/24 |


<br><br>

## 基本設定

PE11の基本場合はこのようになります。
PE12, PE13, PE14もアドレスが違うだけで同様です。

```
!
interface Loopback0
 ip address 192.168.255.11 255.255.255.255
!
!
interface GigabitEthernet1
 mtu 9000
 ip address 192.168.111.11 255.255.255.0
 ip ospf network point-to-point
 ip ospf cost 10
!
!
interface GigabitEthernet2
 mtu 9000
 ip address 192.168.211.11 255.255.255.0
 ip ospf network point-to-point
 ip ospf cost 20
!
router ospf 1
 router-id 192.168.255.11
 nsf ietf
 network 192.168.111.11 0.0.0.0 area 0
 network 192.168.211.11 0.0.0.0 area 0
 network 192.168.255.11 0.0.0.0 area 0
 bfd all-interfaces
!
```

CR1の基本設定はこうなります。CR2も同様です。

```
!
interface Loopback0
 ip address 192.168.255.1 255.255.255.255
!
interface GigabitEthernet1
 mtu 9000
 ip address 192.168.111.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf cost 1
 cdp enable
!
interface GigabitEthernet2
 mtu 9000
 ip address 192.168.112.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf cost 2
 cdp enable
!
interface GigabitEthernet3
 mtu 9000
 ip address 192.168.113.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf cost 3
 cdp enable
!
interface GigabitEthernet4
 mtu 9000
 ip address 192.168.114.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf cost 4
 cdp enable
!
router ospf 1
 router-id 192.168.255.1
 nsf ietf
 passive-interface default
 no passive-interface GigabitEthernet1
 no passive-interface GigabitEthernet2
 no passive-interface GigabitEthernet3
 no passive-interface GigabitEthernet4
 no passive-interface Loopback0
 network 0.0.0.0 255.255.255.255 area 0
 bfd all-interfaces
!
```

<br><br>

## SR-MPLSの設定

ここまでの設定で全てのPE間でループバックアドレス同士のpingが可能になっています。

<br><br>

### 全てのCRとPEでセグメントルーティングを有効にします。

SIDとして利用する番号の範囲をグローバルブロック（SRGB = SR Global Block）と呼びます。

グローバルブロックは同じSRドメイン内で共通にすることが推奨されます。

グローバルブロックのデフォルト値は装置の実装によって違いますので、異機種が混在したときに困らないように決めておいたほうがいいでしょう。

> 参考
>
> シスコのデフォルト値は16000-23999です。

ここでは `20000 - 29999` を使うことにします。

定義したSRGBの情報はIGPで広告されます。

```
!
segment-routing mpls
 !
 global-block 20000 29999
 !
!
```

<br><br>

### 続いて装置を代表するSIDである、ノードSIDを各装置に定義します。

ここでは装置を代表するノードSIDを固定で設定します。

各装置、次のように採番します。

| 装置 | ノードSID | ループバックアドレス |
| ---- | -------- | ------------------ |
| CR1  | 20001    | 192.168.255.1      |
| CR2  | 20002    | 192.168.255.2      |
| PE11 | 20011    | 192.168.255.11     |
| PE12 | 20012    | 192.168.255.12     |
| PE13 | 20013    | 192.168.255.13     |
| PE14 | 20014    | 192.168.255.14     |

`connected-prefix-sid-map` 設定でIPプレフィクスとSIDを対応づけます。

PE11の場合、ループバックのIPアドレスは192.168.255.11なので、これに対応するSIDを20011にするためには次のように設定します。

```
!
segment-routing mpls
 connected-prefix-sid-map
  address-family ipv4
   192.168.255.11/32 index 11 range 1
  exit-address-family
 !
!
```

192.168.255.11/32のアドレス範囲に対して、20000 + インデックス11 = 20011が対応するSIDになります。

IGPで配られるSIDの情報も実態はインデックス番号です。SRGBの先頭の値にインデックスを足すことでSIDを計算します。

ノードSIDは絶対値で指定することも可能です。

PE12は絶対値指定で設定してみます。

```
!
segment-routing mpls
 connected-prefix-sid-map
  address-family ipv4
   192.168.255.12/32 absolute 20012 range 1
  exit-address-family
 !
!
```

どちらの設定方法でも得られる結果は同じです。

全てのPE、CRで同様に設定します。

<br><br>

### 続いてOSPFをSR-MPLSに対応させます。

OSPFでSR-MPLSを有効にすることで、Opaque LSAを使ってSID情報が配信されるようになります。

設定はこれだけです。

```
!
router ospf 1
 segment-routing mpls
!
```

インタフェースに割り当てられているIPプレフィクスに関してSIDが対応付けられ、そのSIDの情報が配信されます。

この動作は従来のMPLSにおいてLDPが実施していることと同じです。
したがってOSPFでSR-MPLSを有効にしたらLDPは不要になります。
ここではLDPが起動しないように明示的に設定することにします。

```
!
no mpls ip
!
```

既存でLDPが動いていて、SR-MPLSにマイグレーションしたいという場合には、その優先順位を設定できます。
次のように設定することでSR-MPLSで生成したラベルが優先されるようになります。

```
!
segment-routing mpls
 !
 set-attributes
  address-family ipv4
   sr-label-preferred
  exit-address-family
 !
```

ループバックインタフェースに付けたIPアドレスは自動的にノードSIDに変換されます。

前述の設定でノードSIDはすでに固定してありますが、ループバックインタフェースを複数作った場合、それもノードSIDになってしまいます。

（ノードSIDが複数あっても困ることはないと思いますが）ループバックインタフェースの設定で明示的にノードSIDにならないようにすることもできます。

たとえばこのように２つのループバックインタフェースを作ったときに、ノードSIDとして利用しない方は`n-flag-clear`を設定します。

```
!
interface Loopback0
 ip address 192.168.255.11 255.255.255.255
!
interface Loopback1
 ip address 192.168.0.111 255.255.255.255
 ip ospf prefix-attributes n-flag-clear
!
```

### Explicit NULLを設定します。

MPLSのデフォルト動作では最終目的地の一つ手前のノードでラベルが剥がされます。
最終目的地のルータで、①ラベルを剥がして、②出てきたパケットの中身をチェックしてルーティング、という２つの作業を実施すると、そのルータの処理が重たくなってボトルネックになりかねません。
一つ前のルータでラベルを剥がしてあげれば作業が分散され、全体的なパフォーマンスが向上するわけですが、それによるデメリットもあります。

ラベルの中には運んでいるパケットのToS情報が格納されていたりします。ラベルを剥ぎ取ってしまうとその情報が消失してしまいます。
剥ぎ取るのではなくNULLラベルを付けて最後までラベルを維持するのがexplicit-nullです。
VPNを使うときにはexplicit-nullを使うようにした方がよいでしょう。

explicit-nullを有効にするには次のように設定します。

```
!
segment-routing mpls
 !
 set-attributes
  address-family ipv4
   explicit-null
  exit-address-family
 !
!
```

<br><br>

## ここまでの設定で状態を確認します。

ここまでの設定で以下の設定を投入しました。

- OSPFを有効にしました
- SR-MPLSを有効にしました
- ノードSIDを固定的に設定しました
- OSPFをSR-MPLS対応に設定しました

### グローバルブロックを確認します

`show segment-routing mpls gb`

```
PE11#show segment-routing mpls gb
LABEL-MIN  LABEL_MAX  STATE           DEFAULT
20000      29999      ENABLED         No
```

設定した20000-29999になっています。

<br>

### ノードSIDを確認します

`show segment-routing mpls connected-prefix-sid-map ipv4`

```
PE11#show segment-routing mpls connected-prefix-sid-map ipv4

               PREFIX_SID_CONN_MAP ALGO_0

    Prefix/masklen   SID Type Range Flags SRGB
 192.168.255.11/32    11 Indx     1         Y

               PREFIX_SID_PROTOCOL_ADV_MAP ALGO_0

    Prefix/masklen   SID Type Range Flags SRGB Source
  192.168.255.1/32     1 Indx     1         Y  OSPF Area 0 192.168.255.1
  192.168.255.2/32     2 Indx     1         Y  OSPF Area 0 192.168.255.2
 192.168.255.11/32    11 Indx     1         Y  OSPF Area 0 192.168.255.11
 192.168.255.12/32    12 Indx     1         Y  OSPF Area 0 192.168.255.12
 192.168.255.13/32    13 Indx     1         Y  OSPF Area 0 192.168.255.13
 192.168.255.14/32    14 Indx     1         Y  OSPF Area 0 192.168.255.14
```

全てのPEルータ、CRルータのSID（のインデックス）が表示されています。
実際のSIDはグローバルブロックの先頭番号20000を加えたものになります。

装置ごとにSRGBの開始番号が異なっているとややこしくなりますので、ドメイン内でグローバルブロックは共通にしておいた方がいいと思います。

<br>

### 隣接関係SID(Adj-SID)を確認します

動的ルーティング(ここではOSPF)で隣接ノードを見つけると、その装置に向いたインタフェースにSIDが採番されます。

`show ip ospf segment-routing adjacency-sid`

```
PE11#show ip ospf segment-routing adjacency-sid

            OSPF Router with ID (192.168.255.11) (Process ID 1)
    Flags: S - Static, D - Dynamic,  P - Protected, U - Unprotected, G - Group, L - Adjacency Lost

Adj-Sid  Neighbor ID     Interface          Neighbor Addr   Flags   Backup Nexthop  Backup Interface
-------- --------------- ------------------ --------------- ------- --------------- ------------------
18       192.168.255.1   Gi1                192.168.111.1   D U
20       192.168.255.2   Gi2                192.168.211.2   D U
```

Adj-Sidの数字はインデックスではありません。MPLSで用いられるラベル値そのものです。

<br>

### MPLSテーブルを確認します

`show mpls forwarding-table`

```
PE11#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
18         Pop Label  192.168.111.1-A  0             Gi1        192.168.111.1
20         Pop Label  192.168.211.2-A  0             Gi2        192.168.211.2
22         No Label   10.0.11.0/24[V]  0             aggregate/L3VPN
20001      explicit-n 192.168.255.1/32 17749         Gi1        192.168.111.1
20002      explicit-n 192.168.255.2/32 19335         Gi2        192.168.211.2
20012      20012      192.168.255.12/32   \
                                       0             Gi1        192.168.111.1
20013      20013      192.168.255.13/32   \
                                       0             Gi1        192.168.111.1
20014      20014      192.168.255.14/32   \
                                       0             Gi1        192.168.111.1

A  - Adjacency SID
```

隣接関係SIDとノードSIDしか見えていません。

PE11からPE13の **物理インタフェース** 192.168.113.13 にpingしてみます。

![PE11からPE13のGig1にping](img/ping_from_pe11_to_pe13_gig1.PNG)

MPLSになっていません。IPv4パケットです。

PE11からPE13の **ループバック** 192.168.255.13 にpingしてみます。

![PE11からPE13のLo0にping](img/ping_from_pe11_to_pe13_lo0.PNG)

こんどはICMP Echo Requestにラベル20013がついていることが分かります。

このようにSR-MPLSではドメイン内の全通信がMPLSになるわけではなく、SRGBの範囲内で割り当てられたノードSIDだけがMPLSになっていることがわかります。

<br><br>

## EVPNを設定します

順番に設定していきます。

![設計パラメータ](img/evpn.parameters.drawio.png)

<br>

### EVPNを有効にします

l2vpn evpnセクションでEVPNに共通の動作を指定します。

```
l2vpn evpn
 replication-type ingress
 mpls label mode per-ce
 router-id Loopback0
!
```

replication-typeの指定は必須です。

![replication-type ingress](img/replication.drawio.png)

PEルータは学習していないMACアドレスを受信したら自分以外の全てのPEにそれを届けなければいけません。
`replication-type`はそのやり方を定義するもので、
ingressは受信したノードでパケットのコピーを作成して全PEにユニキャストで転送する方式です。
マルチキャストで配信できればよいのですが、MPLS網ではそういうわけにいきませんので、事実上ingressの一択です。

<br>

### EVIとBDを定義します

EVIとBDは以下の関係にあります。

![eviとbdの関係](img/evi_bd.drawio.png)

EVIはEVPNを識別する識別子で、L3VPNにおけるVRFに相当するものです。
どのPEにある、どのEVIでイーサネットを形成するか、を設計するものなので、ネットワーク全域を見渡して設計するパラメータです。

ここではPE11, PE12, PE13, PE14で共通のEVIを定義します。

```
l2vpn evpn instance 100 vlan-based
 rd 65000:100
 route-target export 65000:100
 route-target import 65000:100
!
```

instance 100の100は装置内でユニークであれば何番でも構いません。
重要なのはRDの方です。
全てのPEでRD=65000:100としたEVIを作成します。

この装置で学習したMACテーブルを65000:100として他のPEに広告します（export）。

他のPEが65000:100として広告してきたMACテーブルをこのEVIに取り込みます（import）。


```
!
bridge-domain 100
 member evpn-instance 100
!
```

bridge-domain 100の100は装置内でユニークであれば何番でも構いません。

ブリッジドメインからみると、ダウンリンク側として自装置のどのポートがブリッジに参加するか、そしてアップリンク側としてどのEVIがそのブリッジに参加するか、を指定することになります。
ここではまずアップリンク側のEVIをメンバーとして指定しています。

<br>

### ESIの設定

ESIはEVPNに参加するイーサネットを識別するものです。

シングル構成であれば明示的に設定する必要はなく、自動採番で構いません。

マルチホーム接続の場合にはESIを適切に設定することで異なるPE装置間で同一のLANを形成することができます。

![ESI](img/esi.drawio.png)


### CE向け物理ポートの設定


> 注意
>
> ESI-LAGを組むときには、対向装置はLACPをサポートしたものでなければいけません。ここではNexus9000vを利用しています。

PE11とPE12はポートチャネルを作成して、その中に流れるVLANをブリッジドメインに参加させます。

PE13とPE14は物理ポートをブリッジドメインに参加させます。

PE11の設定

```
!
interface Port-channel1
 no ip address
 no negotiation auto
 evpn ethernet-segment 100
  identifier type 0 00.00.00.00.00.00.00.00.11
  redundancy all-active
  df-election wait-time 1
 lacp device-id 0000.0000.0011
 service instance 100 ethernet
  encapsulation untagged
 !
!
interface GigabitEthernet3
 no ip address
 negotiation auto
 channel-group 1
!

!
bridge-domain 100
 member Port-channel1 service-instance 100
 member evpn-instance 100
!
```

PE12の設定はPE11と同じです。

```
!
interface Port-channel1
 no ip address
 no negotiation auto
 evpn ethernet-segment 100
  identifier type 0 00.00.00.00.00.00.00.00.11
  redundancy all-active
  df-election wait-time 1
 lacp device-id 0000.0000.0011
 service instance 100 ethernet
  encapsulation untagged
 !
!
interface GigabitEthernet3
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 1 mode active

!
bridge-domain 100
 member Port-channel1 service-instance 100
 member evpn-instance 100
!
```

Port-channel直下のこの設定がESIを定義したものです。

```
 evpn ethernet-segment 100
  identifier type 0 00.00.00.00.00.00.00.00.11
```

100という数字は特に意味を持っていないと思います（Port-channel直下には一つしか定義できません）。
ESIをtype 0で指定する場合は9バイトの文字列です。
ここではPE11とPE12共に下1オクテットを11としています。

PE13とPE14の設定はこうなります。

```
!
interface GigabitEthernet3
 no ip address
 negotiation auto
 service instance 100 ethernet
  encapsulation untagged
 !
!
bridge-domain 100
 member GigabitEthernet3 service-instance 100
 member evpn-instance 100
!
```

bridge-domainの設定で、ダウンリンク側はGig3の中のservice-instance 100を、アップリンク側はevi 100を指定しています。

ここでは設定していませんが、VLANタグを付けたり、VLAN番号を変換することもできます。

<br>

### BGPの設定

PE同士でiBGP接続し、MAC学習テーブルの情報を交換します。

フルメッシュでの設定を回避するために、CR1とCR2をルートリフレクタとします。

PE11, PE12, PE13, PE14共通設定(PE_NUMBERのところに11, 12, 13, 14を代入)

```
!
router bgp 65000
 bgp router-id 192.168.255.{{ PE_NUMBER }}
 bgp log-neighbor-changes
 bgp graceful-restart
 no bgp default ipv4-unicast
 neighbor 192.168.255.1 remote-as 65000
 neighbor 192.168.255.1 update-source Loopback0
 neighbor 192.168.255.2 remote-as 65000
 neighbor 192.168.255.2 update-source Loopback0
 !
 address-family l2vpn evpn
  neighbor 192.168.255.1 activate
  neighbor 192.168.255.1 send-community both
  neighbor 192.168.255.1 next-hop-self
  neighbor 192.168.255.1 soft-reconfiguration inbound
  neighbor 192.168.255.2 activate
  neighbor 192.168.255.2 send-community both
  neighbor 192.168.255.2 next-hop-self
  neighbor 192.168.255.2 soft-reconfiguration inbound
 exit-address-family
 !
!
```

<br><br>

## ここまでの設定で状態を確認します。

ここまでの設定で以下の設定を投入しました。

- EVPNを有効にしました
- EVIをRD=65000:100として定義しました
- PE11とPE12はPort-channelを作成しました
- PE11とPE12はESIを明示的に設定しました
- ブリッジドメインを定義しました（アップリンクがEVI、ダウンリンクがservice-instance）
- BGPにaddress-family l2vpn evpnを定義しました


<br>

### EVPNのピアを確認します

`show l2vpn evpn peers`

PE11での実行例です。

```
PE11#sh l2vpn evpn peers

EVI    BD    Peer-IP                   Num routes UP time
------ ----- ------------------------  ---------- --------
Global N/A   192.168.255.12            2          02:42:51
100    100   192.168.255.12            2          02:42:49
100    100   192.168.255.13            2          03:34:25
100    100   192.168.255.14            2          03:34:25
```

PE11とPE12はESI-LAGを構成しています。その関係で192.168.255.12が2台見えています。

<br>

### EVIを確認します

`show l2vpn evpn evi`

PE11での実行例です。

```
PE11#show l2vpn evpn evi
EVI   BD    Ether Tag  BUM Label Unicast Label Pseudoport
----- ----- ---------- --------- ------------- ------------------
100   100   0          17        21            Po1:100
```

EVIインスタンス100に関して、BUM(Broadcast Unknown-unicast Multicast)パケットはラベルは17で受信することを期待、ユニキャスト通信はラベル21で受信していることを期待しています。

<br>

### MACアドレステーブルを確認します

`show l2vpn evpn mac`

PE11での実行例です。

CE装置が4台います。全て学習しています。

```
PE11#show l2vpn evpn mac
MAC Address    EVI   BD    ESI                      Ether Tag  Next Hop(s)
-------------- ----- ----- ------------------------ ---------- ---------------
aabb.cc00.0700 100   100   0000.0000.0000.0000.0011 0          192.168.255.12
aabb.cc00.0800 100   100   0000.0000.0000.0000.0011 0          Po1:100
aabb.cc00.0900 100   100   0000.0000.0000.0000.0000 0          192.168.255.13
aabb.cc00.0a00 100   100   0000.0000.0000.0000.0000 0          192.168.255.14
```

PE12で実行するとこのようになります。

```
PE12#show l2vpn evpn mac
MAC Address    EVI   BD    ESI                      Ether Tag  Next Hop(s)
-------------- ----- ----- ------------------------ ---------- ---------------
aabb.cc00.0700 100   100   0000.0000.0000.0000.0011 0          Po1:100
aabb.cc00.0800 100   100   0000.0000.0000.0000.0011 0          Po1:100
aabb.cc00.0900 100   100   0000.0000.0000.0000.0000 0          192.168.255.13
aabb.cc00.0a00 100   100   0000.0000.0000.0000.0000 0          192.168.255.14
```

PE13で実行するとこのようになります。

```
PE13#show l2vpn evpn mac
MAC Address    EVI   BD    ESI                      Ether Tag  Next Hop(s)
-------------- ----- ----- ------------------------ ---------- ---------------
aabb.cc00.0700 100   100   0000.0000.0000.0000.0011 0          192.168.255.12
aabb.cc00.0800 100   100   0000.0000.0000.0000.0011 0          192.168.255.11
aabb.cc00.0900 100   100   0000.0000.0000.0000.0000 0          Gi3:100
aabb.cc00.0a00 100   100   0000.0000.0000.0000.0000 0          192.168.255.14
```

PE14で実行するとこのようになります。

```
PE14#show l2vpn evpn mac
MAC Address    EVI   BD    ESI                      Ether Tag  Next Hop(s)
-------------- ----- ----- ------------------------ ---------- ---------------
aabb.cc00.0700 100   100   0000.0000.0000.0000.0011 0          192.168.255.12
aabb.cc00.0800 100   100   0000.0000.0000.0000.0011 0          192.168.255.11
aabb.cc00.0900 100   100   0000.0000.0000.0000.0000 0          192.168.255.13
aabb.cc00.0a00 100   100   0000.0000.0000.0000.0000 0          Gi3:100
```

PE11とPE12で構成したESI-LAGの配下にいる2台の端末（aabb.cc00.0700とaabb.cc00.0800）は、PE11とPE12で分散されています。

<br>

### ESIの情報を確認します

`show l2vpn evpn ethernet-segment detail`

PE11での実行例です（ESIを明示的に定義しているのはPE11とPE12だけですので、PE13やPE14で実行しても何も表示されません）。

```
PE11#show l2vpn evpn ethernet-segment detail
EVPN Ethernet Segment ID: 0000.0000.0000.0000.0011
  Interface:              Po1
  Redundancy mode:        all-active
  DF election wait time:  1 seconds
  Split Horizon label:    115
  State:                  Ready
  Encapsulation:          mpls
  Ordinal:                0
  RD:                     192.168.255.11:1
    Export-RTs:           65000:100
  Forwarder List:         192.168.255.11 192.168.255.12
```

Forwarder ListにPE11とPE12のアドレスが格納されています。

<br>

### l2vpn evpnアドレスファミリのBGPテーブルを確認します

`show bgp l2vpn evpn`

PE11での実行例です。分かりづらいです。

```
PE11#show bgp l2vpn evpn
BGP table version is 238, local router ID is 192.168.255.11
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:100
 *>   [1][65000:100][00000000000000000011][0]/23
                      ::                                 32768 ?
Route Distinguisher: 192.168.255.11:1
 *>   [1][192.168.255.11:1][00000000000000000011][4294967295]/23
                      ::                                 32768 ?
Route Distinguisher: 192.168.255.12:1
 *>i  [1][192.168.255.12:1][00000000000000000011][4294967295]/23
                      192.168.255.12           0    100      0 ?
 * i                   192.168.255.12           0    100      0 ?
Route Distinguisher: 65000:100
 *>i  [2][65000:100][0][48][AABBCC000900][0][*]/20
                      192.168.255.13           0    100      0 ?
 * i                   192.168.255.13           0    100      0 ?
 *>i  [2][65000:100][0][48][AABBCC000A00][0][*]/20
                      192.168.255.14           0    100      0 ?
 * i                   192.168.255.14           0    100      0 ?
 *>   [3][65000:100][0][32][192.168.255.11]/17
                      ::                                 32768 ?
 * i  [3][65000:100][0][32][192.168.255.12]/17
                      192.168.255.12           0    100      0 ?
 *>i                   192.168.255.12           0    100      0 ?
 *>i  [3][65000:100][0][32][192.168.255.13]/17
                      192.168.255.13           0    100      0 ?
 * i                   192.168.255.13           0    100      0 ?
 *>i  [3][65000:100][0][32][192.168.255.14]/17
                      192.168.255.14           0    100      0 ?
 * i                   192.168.255.14           0    100      0 ?
Route Distinguisher: 192.168.255.11:17
 *>   [4][192.168.255.11:17][00000000000000000011][32][192.168.255.11]/23
                      ::                                 32768 ?
Route Distinguisher: 192.168.255.12:17
 *>i  [4][192.168.255.12:17][00000000000000000011][32][192.168.255.12]/23
                      192.168.255.12           0    100      0 ?
 * i                   192.168.255.12           0    100      0 ?
```
