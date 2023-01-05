
# SRv6

https://www.segment-routing.net/

SRv6 = **S**egment **R**outing over IP**v6** dataplane

A source-routing architecture that seeks the right balance between distributed intelligence and centralized optimization.

（分散インテリジェンスと集中最適化の間の適切なバランスを追求するソース ルーティング アーキテクチャ。Google翻訳）


## ひとことでいうと

新しいアドレス体系を使ってパケットの通り道を制御する仕組みです。

受信した装置が途中経路と出口を指定できるので、ソースルーティングの一種になります。


![fig_1](fig_1.drawio.svg)

R1に着信したパケットがR4から出ていくことを考えます。

IPネットワークの動的ルーティングでは、最短コストに従ってルーティングされますので、コスト設計をどれだけ頑張っても

- R1-R2-R4
- R1-R3-R4
- R1-R2-R3-R4
- R1-R3-R2-R4

の通り道しか実現できません。

SRv6の場合は、着信したR1がどこを通したいかを制御できますので、R1-R2-R3-R2-R4のような通り道も実現可能です。

R3が特別な役割、たとえばフィルタリングやパケットの中身を精査する機能を持っていて、一度そこを通してから外に出したい場合に使えそうです。


## SRv6は○○ではない

**ルーティングプロトコルではありません**。

新しいアドレス体系を運ぶためにISISやOSPF、BGPを拡張しますが、SRv6という動的ルーティングプロトコルがあるわけではありません。

**データを運ぶネットワーク層、トランスポート層ではありません**。

SRv6ではパケットを運ぶためにIPv6が必要です。SR-MPLSではMPLSを使ってパケットを運びます。

SRv6はアドレス体系ですので、それ単独でパケットを運ぶことは動作できません。

## セグメントってなんだ？

セグメントは**番号を採番する対象の機能**を指します。

セグメントというと、ルータで区切られた部分（ブロードキャストが届く範囲）に割り当てるIPアドレスのことを思い浮かべることが多いと思います。

もちろんそれもセグメントです。

セグメントルーティングでは「そこにいるノードにパケットを中継する」という機能に対して番号を採番します。

ルータの中に仮想ルータ（VRF）を作成したのであれば、そのVRFに対して番号を採番します。

その番号に向けてパケットを送信すればVRFにパケットが届くようになる、ということです。

この番号のことをSID(シッド、セグメントID、Segment ID)と呼びます。

SRv6の場合はIPv6のアドレス形式と同じ形式を利用する。

SR-MPLSの場合は単なる整数値を割り当てます。


## SRv6の利用シーン

- 契約に応じて通信する場所を変える（スライシング、5Gのコアでよく言われる機能）

- サービスチェイニング（必要に応じてトラフィックを捻じ曲げてファイアウォールや負荷分散装置等を経由できるので、網に付加価値をつけることができる）

このような利用シーンからメジャーな通信キャリアがSRv6に強い関心を示している。


## SRv6の根底にあるネットワーク構成の考え方

- ファブリックネットワーク

ネットワーク全体を、パケット中継のファブリックとしてとらえる。ネットワーク全体を一つの装置として見てもよい。

ネットワーク構成は自律分散型でありつつ、ファブリックネットワークのように（宛先IPに制約されない）柔軟なトラフィック制御できる。

通常のIPネットワークではインタフェースに付与したコストが最小になる場所を通るが、SRv6はソースルーティングの技術なので、**受信したルータ** がどこを通ってどこから出ていくか、を制御できる。

機器を管理するコントローラは不要。経路計算をセンター集約にすることはできる。


網の内側と、網の外側に分けて考える。

シャーシ型の装置を作っているベンダーはこれが得意。
スーパーバイザー = コントローラ
ラインカード = ルータ(やスイッチ)

ファブリックに着信した通信は、ファブリックの中を通ったあと、別の出口から出ていく。

ファブリックネットワークの作り方には時代によって流行りがある。

現在の主流はこれら。
- Cisco ACI  ・・・データプレーンはVXLAN、独自コントロールプレーン（APIC）
- MPLS/BGP VPN　・・・データプレーンMPLS、コントロールプレーンにLDP, BGP

注目株
- SRv6　・・・データプレーンにIPv6を使ったセグメントルーティング
- SR-MPLS　・・・データプレーンにMPLSを使ったセグメントルーティング　★富士通のデータセンターで採用

あまり流行らなかったもの
- Cisco FabricPath (レイヤ2ブリッジの拡張)
- Cisco SD-Access (SD-LAN)　・・・データプレーンはVXLAN、コントロールプレーンはLISP

SRv6はデータプレーンにIPv6を使うので、IPv6が持つ多くの機能を流用できるので、SRのために特別なプロトコルを作る必要がなく、シンプルなこと。

IPv6には拡張ヘッダがあって、それらを活用する。
どこを通れ、という司令はHop-by-Hopヘッダを利用する。
L3VPNの場合は、昔ながらのMPLS-BGP VPNにちょっとだけ手を加えるだけ。

SRv6で必要になる知識はこれら。

- IPv6
- ISIS or OSPFv3（IPv6ルーティング）
- BGP（L3VPN、EVPN）
- SR

企業向けのネットワークインテグレーションをやっていると、あまり馴染みがないものが多いのが難点。

## あらためてセグメントとは

セグメントと聞いてイメージするのは、ルータとルータの間にある部分、ブロードキャストが届く範囲の部分。
それもセグメントの一つ。

ファブリック型のネットワークを作るときに、ファブリック内のアドレスは重要ではない。
端末が存在する末端の部分には当然IPアドレスが必要だが、その情報を中継する役割の部分にIPアドレスはいらない。

その昔、PPP(Point-to-Point Protocol)でWANを接続したときには `ip unnumbered ethernet0` というようにアドレスを割り当てないことが多かった。

端末が存在しないファブリック内部において、番号の割当が必要なのはこの２つ。

- ルータそのもの
- ルータが持つインタフェース

番号はIPアドレスである必要はない。

ルーティングプロトコルのIS-ISはまさにこの考え方で、ルータ本体に番号を付与すれば、インタフェースのアドレスは自動で採番される。

この考え方で番号を割り当てるのが「セグメントルーティング」

網内で採番するのはIPアドレスである必要はないよね、ということ。

とはいってもIPの世界から完全に離れてしまうとこれまで培ってきたプロトコル群との相性が悪くなってしまうので、IPv6のアドレスフォーマットを流用するのがSRv6

ルータ本体にまとまった量のセグメントIDを割り当てて、必要に応じてその中から採番していく。

ルータ自身に持たせるセグメントIDの範囲をロケータという。

物理インタフェースに対しては、ロケータの中から切り出して採番していく。

全体像としては、各ルータがセグメントのブロック（ロケータ）を持っているようなイメージ

フォマットがIPv6と同じなので、IPv6をルーティングできるプロトコルで流通させてしまえば、そのセグメントへの到達性は簡単に確保できる。


## セグメントのフォーマット

IPv6と同じ形をとる。このおかげで既存のルーティングプロトコルと相性がよい。

```
[      Locator      ][      Function      ]
[ Block   ][ Node   ]
  40 bit     24 bit           64 bit
```

ロケータ、すなわちそのルータ自身にユニークに採番される部分は64ビット。IPv6のネットワーク部（プレフィクス部）に相当。

ルータに対して割り当てるものなのでセグメントIDの先頭64ビットを見れば、どのルータが保有しているものなのかがわかる。

ロケータの内部はBlockとNodeに分かれる。通常Blockは40ビット、Nodeは24ビットになる。

SRv6を構成する網の中でBlock部分は統一しなければならない。

128ビットのセグメントIDのうち前半64ビットがロケータなので、残った64ビット、IPv6でいうところのホスト部の中からルータ自身が必要に応じてセグメントIDを自由に採番していく。

ルータ自身がセグメントIDを割り当てる対象は大きくは２つ。

1. 自足のインタフェース
2. そのルータ自身が持つサービス　よく使われるのはVPNのVRF

ルータ自身がセグメントを割り当てる対象は物理インタフェースに限定されないので `Function` と呼ぶ。

## Function

FunctionはルータがセグメントIDを採番する対象で、そのID宛てにパケットが届いたときにどのように振る舞うかも合わせて定義する。

FunctionはRFC8986で標準化されている。

RFC8986 Segment Routing over IPv6 (SRv6) Network Programming

https://datatracker.ietf.org/doc/rfc8986/

よく目にするFunctionはこれら。

- End そのルータ自身に届くためのFunction

- End.X そのルータ自身に届いた後、別の隣接ノードに中継する

- End.DT4 そのルータ自身に届いた後、L3VPNのIPv4ルーティングテーブルを検索して中継する

- End.DT6 そのルータ自身に届いた後、L3VPNのIPv6ルーティングテーブルを検索して中継する


## End.X

End.Xは転送に利用する。

次に転送する先のインタフェースとIPv6アドレスが分かる。

```bash
fx201-p#show segment-routing srv6 sid detail

SID                         Function     Context                                             Owner  State
--------------------------  -----------  --------------------------------------------------  -----  ---------
3ffe:201:0:1:42::           End.X        [Port-channel 1020000, Link-Local]                  IS-IS  InUse
  Locator : prefix1
  Nexthop : fe80::280:bdff:fe4c:b2b2
  Link-ID : 7
  OUT-RFID: 65537
  Created : Wed Dec 14 18:11:20 2022 (02w5d16h ago)
3ffe:201:0:1:43::           End.X (PSP)  [Port-channel 1020000, Link-Local]                  IS-IS  InUse
  Locator : prefix1
  Nexthop : fe80::280:bdff:fe4c:b2b2
  Link-ID : 7
  OUT-RFID: 65537
  Created : Wed Dec 14 18:11:20 2022 (02w5d16h ago)
```

固定でSIDを作るには次のコマンドを使う。

```bash
local-sid <SID> action end.x <送信インターフェイス名> <Next-hop> [psp]
```


## End.DT4

L3VPNなのでiBGPを使ってエッジルータ同士が情報交換する。

このとき交換するのはロケータの情報と **ラベル** の情報。昔ながらのMPLS-VPNの仕組みをそのまま流用している。

L3VPNの情報はshow ip bgp vpnv4 all で確認できる。

```
fx201-pe1#show ip bgp vpnv4 all 220.0.1.0

Route Distinguisher: 1:1 (1)
BGP routing table entry for 220.0.1.0/24
  Not advertised to any peer
  Local
    3ffe:220:1::1 (metric 30) from 3ffe:220:1::1 (220.0.0.1)
      Origin incomplete, metric 0, localpref 100, valid, internal, best, installed
      Extended Community: RT:1:1
      Original RD:1:1
      BGP Prefix-SID: SRv6 L3VPN 3ffe:220:1:1:: (L:40.24, F:16.0, T:16.64) End.DT4
      Local Label: no label
      Remote Label: 1120
      Path Identifier (Remote/Local): /0
      Last update: Wed Dec 14 18:05:34 2022
```

`BGP Prefix-SID: SRv6 L3VPN 3ffe:220:1:1:: (L:40.24, F:16.0, T:16.64) End.DT4`

という部分に注目。Prefix-SIDはロケータのことで、ルータそのものを指す。

`3ffe:220:1:1::` は対向するPEルータ、つまりこの情報を教えてくれたルータを指している。

- L:40.24 の意味

> マニュアルから引用
> SRv6 SID Structure Sub-Sub-TLV の LocBlock len, LocNode len を示しています

ロケータのBlock部が40ビット、Node部が24ビットという意味。2つ合わせて40+24=64ビットがロケータの長さ。
どのメーカーのSRv6ルータでもこうなっているはず。

- F:16.0 の意味

> マニュアルから引用
> SRv6 SID Structure Sub-Sub-TLV の Function len, Argument len を示しています

Function部の長さが16ビット、引数となるArgumentの長さが0ビット、という意味。2つ合わせて16+0=16ビットがFunction部の長さ。
この部分はメーカーによって違うかもしれない。

- T:16.64 の意味

> マニュアルから引用
> SRv6 SID Structure Sub-Sub-TLV の Trans len, Trans Offset を示しています

RFC9252(BGP Overlay Services Based on Segment Routing over IPv6) を読まないと、この意味はわからない。

RFC9252から引用。

```
   Transposition Length (1 octet):
      This field is the size in bits for the part of the SID that has
      been transposed (or shifted) into an MPLS Label field.

   Transposition Offset (1 octet):
      This field is the offset position in bits for the part of the SID
      that has been transposed (or shifted) into an MPLS Label field.
```

BGPを使ってVPNの経路情報を交換するときには、VPNを識別する情報としてMPLSのラベルを付与する。

MPLSのラベルの長さは20ビット。

セグメントIDの長さは128ビット。

当然収まりきらないので工夫して格納する。ここがすごく分かりづらいところ。

128ビットのSIDのうち、どの部分からどの部分までを20ビットのラベル部に格納したのかを表すのがTrans lenとTrans Offset。

T:16.64は、SIDの先頭64ビットから16ビットを切り出したもの、という意味になる。これはちょうどFunction部を表している。

- Remote Label: 1120 の意味

対向ルータが送ってきたラベルの値のこと。

10進数の `1120` は16進数では `0x 0460`、2進数では `0b 0000 0100 0110 0000` になる。

20ビットの器に入っているので正確には `0x 00460` = `0b 0000 0000 0100 0110 0000` となる。

T:16.64と指定された通り、128ビットのSIDの先頭64ビットから16ビット分がここに格納されていることになるので、先頭から16ビット分を取り出す。
すなわち下4ビットを破棄すると、2進数で `0000 0000 0100 0110`、16進数で`0x0046` となる。

IPv6のアドレスは連続するゼロを省略するのでFunction部は`0x46`ということになる。

ロケータとラベル情報からSIDを組み立てると、ロケータの`3ffe:220:1:1::`にラベルの`0x46`を連結して`3ffe:220:1:1:46::`がEnd.DT4のSIDになる。

IOS-XRの場合。

```
RP/0/RP0/CPU0:PE04#show bgp vrf vrf1 192.168.5.0/24
Thu Dec 15 14:26:10.464 JST
BGP routing table entry for 192.168.5.0/24, Route Distinguisher: 100:1
Versions:
  Process           bRIB/RIB  SendTblVer
  Speaker                  31           31
Last Modified: Dec 15 14:23:30.905 for 00:02:39
Paths: (2 available, best #1)
  Advertised to CE peers (in unique update groups):
    192.168.6.6
  Path #1: Received by speaker 0
  Advertised to CE peers (in unique update groups):
    192.168.6.6
  65005
    2001:db8::3 (metric 30) from 2001:db8::1 (192.168.255.3)
      Received Label 0x420
      Origin IGP, metric 0, localpref 100, valid, internal, best, group-best, import-candidate, imported
      Received Path ID 0, Local Path ID 1, version 31
      Extended community: RT:100:1
      Originator: 192.168.255.3, Cluster list: 0.0.0.1
      PSID-Type:L3, SubTLV Count:1
       SubTLV:
        T:1(Sid information), Sid:2001:db8:0:3::, Behavior:19, SS-TLV Count:1
         SubSubTLV:
          T:1(Sid structure):
      Source AFI: VPNv4 Unicast, Source VRF: vrf1, Source Route Distinguisher: 100:1
```

Trans LenとTrans Offsetが不明だが、おそらくFITELnetと同じでT:16.64のはず。

`Received Label 0x420` からこのプレフィクスに紐づいたラベル値は0x420 = 0x00420であることがわかる。

頭から16ビット分を取り出す、すなわち下4ビットを破棄して0x0042がFunction部ということになる。

`T:1(Sid information), Sid:2001:db8:0:3::, Behavior:19, SS-TLV Count:1` からロケータは2001:db8:0:3であることがわかる。

ロケータとラベルの情報から、この経路に紐づくSIDは `2001:db8:0:3:42` となる。


RFC9252 BGP Overlay Services Based on Segment Routing over IPv6 (SRv6)

```
5.1.  IPv4 VPN over SRv6 Core

   The MP_REACH_NLRI over SRv6 core is encoded according to IPv4 VPN
   unicast over IPv6 core defined in [RFC8950].

   The label field of IPv4-VPN NLRI is encoded as specified in [RFC8277]
   with the 20-bit Label Value set to the whole or a portion of the
   Function part of the SRv6 SID when the Transposition Scheme of
   encoding (Section 4) is used; otherwise, it is set to Implicit NULL.
   When using the Transposition Scheme, the Transposition Length MUST be
   less than or equal to 20 and less than or equal to the FL.

   The SRv6 Service SID is encoded as part of the SRv6 L3 Service TLV.
   The SRv6 Endpoint Behavior SHOULD be one of these: End.DX4, End.DT4,
   or End.DT46.
```

Transposition Scheme of encoding (Section 4) というのはこの部分。

```
4.  Encoding SRv6 SID Information

   The SRv6 Service SID(s) for a BGP service prefix is carried in the
   SRv6 Services TLVs of the BGP Prefix-SID attribute.

   For certain types of BGP Services, like L3VPN where a per-VRF SID
   allocation is used (i.e., End.DT4 or End.DT6 behaviors), the same SID
   is shared across multiple NLRIs, thus providing efficient packing.
   However, for certain other types of BGP Services, like EVPN Virtual
   Private Wire Service (VPWS) where a per-PW SID allocation is required
   (i.e., End.DX2 behavior), each NLRI would have its own unique SID,
   thereby resulting in inefficient packing.

   To achieve efficient packing, this document allows either 1) the
   encoding of the SRv6 Service SID as a whole in the SRv6 Services TLVs
   or 2) the encoding of only the common part of the SRv6 SID (e.g.,
   Locator) in the SRv6 Services TLVs and the encoding of the variable
   (e.g., Function or Argument parts) in the existing label fields
   specific to that service encoding.  This later form of encoding is
   referred to as the Transposition Scheme, where the SRv6 SID Structure
   Sub-Sub-TLV describes the sizes of the parts of the SRv6 SID and also
   indicates the offset of the variable part along with its length in
   the SRv6 SID value.  The use of the Transposition Scheme is
   RECOMMENDED for the specific service encodings that allow it, as
   described further in Sections 5 and 6.
```

Transposition Schemeは転置スキームと訳すのかな。

End.DT4のようにPer-CEやPer-VRFでSIDを割り当てるサービスでは、プレフィクスに対して同じSIDを割り当てることになるので、同じ情報を何度も繰り返し送信すのは無駄になる。
転置スキームを使って一度送ったらそれを再利用する。

```
3.2.1.  SRv6 SID Structure Sub-Sub-TLV

   SRv6 Service Data Sub-Sub-TLV Type 1 is assigned for the SRv6 SID
   Structure Sub-Sub-TLV.  The SRv6 SID Structure Sub-Sub-TLV is used to
   advertise the lengths of the individual parts of the SRv6 SID, as
   defined in [RFC8986].  The terms Locator Block and Locator Node
   correspond to the B and N parts, respectively, of the SRv6 Locator
   that is defined in Section 3.1 of [RFC8986].  It is carried as Sub-
   Sub-TLV in the SRv6 SID Information Sub-TLV.

    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | SRv6 Service  |    SRv6 Service               | Locator Block |
   | Data Sub-Sub  |    Data Sub-Sub-TLV           | Length        |
   | -TLV Type=1   |    Length                     |               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | Locator Node  | Function      | Argument      | Transposition |
   | Length        | Length        | Length        | Length        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | Transposition |
   | Offset        |
   +-+-+-+-+-+-+-+-+

                  Figure 5: SRv6 SID Structure Sub-Sub-TLV

   SRv6 Service Data Sub-Sub-TLV Type (1 octet):
      This field is set to 1 to represent the SRv6 SID Structure Sub-
      Sub-TLV.

   SRv6 Service Data Sub-Sub-TLV Length (2 octets):
      This field contains a total length of 6 octets.

   Locator Block Length (1 octet):
      This field contains the length of the SRv6 SID Locator Block in
      bits.

   Locator Node Length (1 octet):
      This field contains the length of the SRv6 SID Locator Node in
      bits.

   Function Length (1 octet):
      This field contains the length of the SRv6 SID Function in bits.

   Argument Length (1 octet):
      This field contains the length of the SRv6 SID Argument in bits.

   Transposition Length (1 octet):
      This field is the size in bits for the part of the SID that has
      been transposed (or shifted) into an MPLS Label field.

   Transposition Offset (1 octet):
      This field is the offset position in bits for the part of the SID
      that has been transposed (or shifted) into an MPLS Label field.
```

SIDのFunction部の情報を送るときに、20ビットのMPLSのラベル情報に置換する際のビットシフトの量をTransposition Lengthで指定する。
ここが4になっているはず。











## FunctionのSIDに到達するための経路は？

IPv6のルーティングテーブルを検索すれば出てくる。

個々のSIDに関する情報を配信しているわけではないので、ロンゲストマッチのルールに従って最後はロケータにたどり着くはず。

```
fx201-pe1#show ipv6 route 3ffe:220:1:1:46::

Routing entry for 3ffe:220:1:1::/64
  Known via "isis", distance 115, metric 30, best, redistributed
  Last update 00:00:01 ago

  fe80::280:bdff:fe4d:5e10 (reachable by fe80:2723::/64), port-channel1010000, RD 0:0, System VRF-ID 0, NHD LINK fe80:2723::280:bdff:fe4d:5e10 (25), refcnt 4
  fe80::280:bdff:fe4c:b2a3 (reachable by fe80:2726::/64), port-channel3010000, RD 0:0, System VRF-ID 0, NHD LINK fe80:2726::280:bdff:fe4c:b2a3 (22), refcnt 3
```

## ローカルSIDは再配布される？

静的に決めたSIDはどうなる？




## 疎通確認の方法


IOS-XRの場合は、ポリシー名を指定してping、tracerouteを打てる。

```
RP/0/RP0/CPU0:PE04#ping segment-routing srv6 policy name ?
  WORD  Srv6 TE configured or auto-generated policy name

RP/0/RP0/CPU0:PE04#traceroute segment-routing srv6 policy name a ?
  flowlabel  flowlabel of the packet
  maxttl     maximum time to live
  minttl     minimum time to live
  numeric    Numeric display only
  port       port number
  priority   priority of the packet
  probe      probe count
  reduced    ping segment routing policy with reduced option
  source     source address or interface
  timeout    timeout value in seconds
  verbose    verbose output
  vrf        vrf table for the route lookup
  <cr>
```

Path Tracingという機能が実装されると、どこを通っているかが分かるようになる。

SIDへのpingはできない。

```
f220-pe2#show segment-routing srv6 sid

SID                         Function     Context                                             Owner  State
--------------------------  -----------  --------------------------------------------------  -----  ---------
3ffe:220:1:1:40::           End                                                              IS-IS  InUse
3ffe:220:1:1:41::           End (PSP)                                                        IS-IS  InUse
3ffe:220:1:1:46::           End.DT4      '1'                                                 BGP    InUse
3ffe:220:1:1:47::           End.DT4      '2'                                                 BGP    InUse
3ffe:220:1:1:48::           End.DT6      '1'                                                 BGP    InUse
3ffe:220:1:1:49::           End.DT6      '2'                                                 BGP    InUse
3ffe:220:1:1:4a::           End.X        [Port-channel 1020000, Link-Local]                  IS-IS  InUse
3ffe:220:1:1:4b::           End.X (PSP)  [Port-channel 1020000, Link-Local]                  IS-IS  InUse
3ffe:220:1:1:4c::           End.X        [Port-channel 1010000, Link-Local]                  IS-IS  InUse
3ffe:220:1:1:4d::           End.X (PSP)  [Port-channel 1010000, Link-Local]                  IS-IS  InUse
f220-pe2#ping ipv6 3ffe:220:1:1:40::
Sending 5, 100-byte ICMP Echos to 3ffe:220:1:1:40::(3ffe:220:1:1:40::), timeout is 2 seconds:
*** No route to host.
.*** No route to host.

Success rate is   0 percent (0/2)

f220-pe2#ping 3ffe:220:1:1:41::
Sending 5, 100-byte ICMP Echos to 3ffe:220:1:1:41::(3ffe:220:1:1:41::), timeout is 2 seconds:
*** No route to host.

Success rate is   0 percent (0/1)
```


## OAM (Operation and Maintenance)

ファブリックネットワークは、全体を一つの装置として見るので、個々の装置の管理ではなく、複数の装置を同時に扱う必要がある。
パケットが入ってきた場所と、出ていく場所は異なる装置なのが通常なので、その両方の装置をケアしなければいけない。

ベンダー独自技術の場合はコントローラが付属する。Cisco ACIの場合はAPICがコントローラ。

オープン技術を採用する場合には、マルチベンダ製品を扱うオーケストレータを使う場合が多い。

Cisco NSO(Network Services Orchestrator)

JuniperはCSO(Contail Service Orchestration)

ただ、現場作業時にNSOを持ち込むのはオーバースペックなので、簡易なツールが必要。

pyATSを使って各種操作を自動化する。

できること

- ファブリック内のSIDを全て収集

- パラメータシートからのVPNプロビジョニング

- end-to-endの疎通確認


## 要望

機能追加

- ping srv6 <ポリシー> の実装
- path-tracing の実装
- API(RESTCONF/YANGモデル)の実装
- gRPCを使ったdial-in/outでのテレメトリ送信
