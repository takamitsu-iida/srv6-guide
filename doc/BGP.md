
# BGP


## End.DT4ファンクション


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
