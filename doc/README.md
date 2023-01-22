
# SRv6

https://www.segment-routing.net/

SRv6 = **S**egment **R**outing over IP**v6** dataplane

A source-routing architecture that seeks the right balance between distributed intelligence and centralized optimization.

（分散インテリジェンスと集中最適化の間の適切なバランスを追求するソース ルーティング アーキテクチャ。Google翻訳）

<!--
CiscoのSRv6のFCS(First Customer Shipping)は2019年12月です。
LinuxがSRv6をサポートしているので、LinuxベースのNFVは今後増えてきます。
    - Snort
    - SERA iptables
    - nftables
    - pyroute2
    - netfilter
    - FD.io VPP

Classifier
    - ENEA https://www.enea.com/solutions/dpi-traffic-intelligence/
        組み込み用

速度
    - Cisco Nexusは400GをラインレートでSRv6中継できる

チップベンダー
    - Barefoot Networks 2019年にIntelに買収された
    - Broadcom

NIC
    - Mellanox
    - Intel

NFV Partner
    - TRENDMICRO
    - ENEA

-->

<br>

## ひとことでいうと

新しいアドレス体系を使ってパケットの通り道を制御する仕組みです。

パケットを受信した装置は、その新しいアドレス体系（番号）を使って途中経路や出口を指定できるので、ソースルーティングの一種になります。

たとえばR1に着信したパケットがR4から出ていくことを考えます。

![fig_1_1](img/fig_1_1.drawio.png)

IPネットワークの動的ルーティングでは最短コストに従ってルーティングされますので、コスト設計をどれだけ頑張っても

- R1-R2-R4
- R1-R3-R4
- R1-R2-R3-R4
- R1-R3-R2-R4

の4通りしか実現できません。

一方でSRv6の場合は、着信したR1がどこを通したいかを（新しい番号を使って）制御できますので

- R1-R2-R3-R2-R4

![fig_1_2](img/fig_1_2.drawio.png)

のような、同じノードを2回通るような経路も実現可能です。

これを応用すると、たとえばフィルタリングやパケットの中身を精査する機能を持っている装置に通してから外に出したい場合に使えそうです。

![fig_1_3](img/fig_1_4.drawio.png)

このような最短コストによらない経路制御を、シンプルに、少ないプロトコルで実現するのがSRv6です。

<br>

## SRv6は○○ではない

SRv6は**ルーティングプロトコルではありません**

新しいアドレス体系を扱うためにISISやOSPF、BGPを拡張しますが、SRv6という動的ルーティングプロトコルがあるわけではありません。

SRv6はデータを運ぶ**ネットワーク層、トランスポート層ではありません**

SRv6はアドレス体系ですので、それ単独でパケットを運ぶことは動作できません。
SRv6ではIPv6を、SR-MPLSではMPLSを使ってパケットを運びます。

<br>

## セグメントってなんだ？

セグメントルーティングという言葉を聞いたときに思い浮かぶ疑問の一つがこれではないでしょうか。

セグメントは**番号を採番する対象の機能**のことを指します。

IPネットワークでセグメントといえば、ルータで区切られた部分（ブロードキャストが届く範囲）に割り当てるサブネットを思い浮かべることが多いと思います。
セグメントルーティングではそれを「その場所にいるノードにパケットを中継する機能」と考え、その機能に対して番号を採番します。

ルータの中に仮想ルータ（VRF）を作成したのであれば、そのVRFに対して番号を採番します。
その番号に向けてパケットを送信すればVRFにパケットが届くようになる、ということです。
この番号のことをSID(=Segment ID、シッド、セグメントID)と呼びます。
SRv6においてSIDはIPv6のアドレス形式と同じ形を利用します。SR-MPLSの場合は単なる整数値を使います。

採番対象になる代表的な機能については標準化されて名称がついています（この後紹介します）。

<br>

## SRv6のネットワーク構成の考え方

SRv6はファブリックネットワークです。

![fig_2](img/fig_2.drawio.png)

ネットワーク全体を一つの装置（ファブリック）と捉え、着信したパケットがファブリックのどこかを通って出ていく、と考えます。
したがってネットワークの内側と外側、入り口と出口は明確に分けて考える必要があります。

ファブリックネットワークの作り方には時代によって流行り廃りがあります。

現在の主流はこれらです。

- Cisco ACI  ・・・データプレーンはVXLAN、独自コントロールプレーン（APIC）
- MPLS/BGP VPN　・・・データプレーンMPLS、コントロールプレーンにLDP, BGP

注目株がこちら。

- SRv6　・・・データプレーンにIPv6を使ったセグメントルーティング
- SR-MPLS　・・・データプレーンにMPLSを使ったセグメントルーティング

SRv6とSR-MPLSは現在主流になっているACIやMPLS/BGP VPNよりも、よりオープンで、よりシンプルに実現できます。

残念ながら、あまり普及しなかったものもあります。

- Cisco FabricPath (レイヤ2ブリッジの拡張)
- Cisco SD-Access (SD-LAN)　・・・データプレーンはVXLAN、コントロールプレーンはLISP

シャーシ型の装置を作っているベンダーは、ファブリックネットワークも得意です。
シャーシ型の装置はスーパーバイザーと呼ばれる装置を管理するモジュールと、装置内部でパケットを中継するファブリックモジュール、外部に接続インタフェースを提供するラインカードで構成されます。

![fig_3](img/fig_3.drawio.png)

それらモジュールをそれぞれ単独の装置にして、分散配置すればファブリックネットワークが構成できます。
このようにして作られたベンダー独自のファブリックネットワークには「コントローラ」と呼ばれる装置がつきものです。

一方で、SRv6にコントローラはありません。
各装置が自律的に動作する分散型のアーキテクチャを採用した、専用コントローラが存在しないファブリック型ネットワークです。

SRv6のデータプレーンはIPv6です。
入り口ルータは受け取ったパケットを新しいIPv6パケットで包み込んで出口ルータ目掛けて送信します。
このとき途中経路を指定したり、回線の状況に応じて最適な経路を選択することもできます。

> **Note**
>
> IPv6は何を運んでいるかを「Next Header」フィールドで示します。
> Next Headerが4ならIPv4、41ならIPv6です。

<br>

> **Note**
>
> EVPNでイーサネットフレームを運ぶ場合のNext Headerは143です。
>
> 参照 RFC 8986 Segment Routing over IPv6 (SRv6) Network Programming
>
> 参照 https://www.iana.org/assignments/protocol-numbers/
>
> 古い実装では59(IPv6-NoNxt)になっているかもしれません。
> IOS-XRは`segment-routing srv6 encapsulation evpn next-header`で変更できます。

<br>

> **Note**
>
> SRv6において、機器を管理するコントローラは存在しませんが、経路計算を1台の装置に集約にすることはできます。

<br>

## SRv6の構成要素

ファブリックネットワークですので、入り口と出口が明確になる構成に向いています。

典型的な構成は以下の通りです。

- IPv4 over SRv6
- L3VPN over SRv6
- EVPN over SRv6

![fig_3_2](img/fig_3_2.drawio.png)

このような構成をシンプルに、少ないプロトコルで実現します。
トンネルインタフェースは不要です。MPLSもVXLANも不要です。
イーサだろうとIPv4だろうとIPv6だろうと、全てSRv6だけで運べます。
加えて、運ぶ通信によって通る場所を変えるように設計できるのがSRv6の魅力です。

<br>

## 必要な知識

SRv6を理解するうえで必要になる知識はこれらです。

- IPv6
- ISIS
- BGP
- VPN（L3VPN、EVPN）

企業向けのネットワークインテグレーションではあまり馴染みがないかもしれません。

![fig_4](img/fig_4.drawio.png)

ベースになるネットワークでは網内の全域でIPv6が必要です。

そのためのルーティングプロトコルにはISISが用いられます。
IPv6をルーティングするだけならOSPFv3でもよいのですが、SRv6で導入される新しいアドレス体系と相性が良いのはISISです。
ベンダー各社のSRv6実装状況を鑑みてもルーティングプロトコルはISISの一択になります。

VPNを構成するエッジ・エッジ間での情報交換はiBGPが用いられます。
iBGPはフルメッシュでピアリングしなければいけませんので、通常はルートリフレクタを導入して管理を容易にします。

> **Note**
>
> VPNを使わなくてもIPv4 over SRv6としてエンド・エンドの通信を実現できます。
> ですが、L3VPNとして構成した方が分かりやすいように思います。

<br>

## ルータが持つセグメント

ルータは２つの機能を必ず持っています。

1. 自分宛の通信を処理する機能
2. 隣接する装置にパケットを中継する機能

IPネットワークを設計するときのベストプラクティスでは、すべてのルータにループバックインタフェースを作成して、装置を代表するIPアドレスをそのループバックに割り当てます。
IPv4であれば/32、IPv6であれば/128のアドレスをループバックに割り当てて、動的ルーティングでその情報を配信します。
ルータの識別子、たとえばbgpやISIS、OSPFのルータIDとしてこのアドレスが使われるほか、
実際の通信、たとえばiBGPでピアリングするときのアドレスや、トンネルの送信元アドレスとしてループバックのアドレスが利用されます。

![node_sid](img/node_sid.drawio.png)

SRv6では自装置宛の通信を処理する機能にEndという名称が付与されており、SIDの採番対象となっています。
ノード（装置自身）への通信を処理する機能ですからノードSID(Node-SID)と呼びます。

- 装置を代表するアドレス = ループバックインタフェースのIPアドレス
- 装置を代表するSID = SRv6のEnd機能に対するSID = ノードSID

と考えてよいでしょう。

ルータは自足のインタフェースの先にいるノードにパケットを中継します。
すべてのルータがこの動作をすることで、ホップバイホップでパケットが中継されていくわけですが、SRv6ではこの機能にEnd.Xという名称が付いています。
もちろんSIDを採番する対象で、ノードが持っている足の数だけEnd.XのSIDを割り当てます。これを隣接関係SID（Ajacency SID、Adj-SID）と呼びます。

> **Note**
>
> ざっくり、装置のループバックがnode-SID、特定のインタフェースがadj-SID、と考えていいと思います。

ルータのどの機能にどのSIDを割り当てるか、その方法は大きく分類すると２通りで、一つは人間が決める方法、もう一つは動的に決める方法です。

Endは装置を代表するSIDですから、なるべく固定的に採番されていた方が管理の観点でメリットが大きいです。
IOS-XRではノード部が1になっているSIDが自動で割り当てられます。

一方、End.Xは隣接ノードを見つけたときに自動で決めたほうがいいでしょう。
隣接ノードは障害の発生や構成変更で常に存在するとは限りませんので、見つけ次第、自動で採番する方が理にかなっています。通常これはISISの仕事です。

ルータには上記２つの機能に加えて、VPNを収容することもあります。
セグメントルーティングが画期的なのは、VPNに対しても同じアドレス体系で番号を付与できることです。
どこを通って、どこにたどり着け、という制御をかける際に、全てを同じ番号体系で指し示せるのです。

どこを通れ、は装置を代表するノードSID、どのインタフェースを通れ、はEnd.XのSID、どこにたどり着け、はVPNに対応するSIDを指定します。

<br>

> **Note**
>
> SRv6では、VPNを識別する番号がIPv6アドレスと同じ形式ですので、途中経路上の装置でもVPNを識別できることになります。
> これを利用すれば途中経路上の装置でVPN単位にフィルタや帯域制御をかけられます。

<br>

## セグメントIDの形式

SRv6のセグメントIDはIPv6アドレスと同じ形式を取ります。
IPv6では前半64ビットがネットワーク部、後半64ビットがホスト部になっていますが、SRv6では前半をLocator（ロケータ）、後半をFunction（ファンクション）と呼びます。
ロケータはそれがどの装置に存在するものなのかを表し、ファンクションはその装置の中のどの機能なのかを表します。

![fig_5](img/fig_5.drawio.png)

SRv6のロケータはさらにブロック部とノード部に分かれます。通常ブロック部は40ビット、ノード部は24ビットです。
（40ビットというと16の倍数ではないので、IPv6のアドレス表記とは相性が悪いのですが、そう決まっているので仕方ありません）。
SRv6を構成するネットワーク内でブロック部は共通にします。

たとえば 2001:0db8:0000:/48 を例に考えてみます。

<br>

> **Note**
>
> 2001:0db8で始まるアドレスは文書記載用に予約されたアドレスです。

<br>

> **Note**
>
> プロバイダーからIPv6アドレスの払い出しを受ける際の最大サイズは一般的に/48です。

<br>

> **Note**
>
> アドレスにかかる費用
> https://www.nic.ad.jp/ja/ip/member/fee-table-2012.html#fee-table


<br>

先頭の40ビットはブロック部になりますので、SRv6を構成する全ルータのロケータで共通でなければいけません。
/48のサイズでアドレスの払い出しを受けていれば、このルールはすでに満たしています。
この例では 2001:0db8:00 がブロック部になります（16進表記のXXが5個で40ビットです）。

ノード部はルータごとに変わります。ノード部を1から連番で割り当てるなら、各装置のロケータはこうなります。

- R1のロケータは 2001:0db8:00 + 00:0001
- R2のロケータは 2001:0db8:00 + 00:0002
- R2のロケータは 2001:0db8:00 + 00:0003
- R2のロケータは 2001:0db8:00 + 00:0004

![fig_6](img/fig_6.drawio.png)

<br>

> **Note**
>
> ここでのaはロケータに付与した名前です。
> FlexAlgoを導入するとロケータは複数必要になりますので、採番ルールと同時に名前付けのルールも意識する必要があります。

<br>

## IPv6アドレスとロケータの採番

IPv6の場合、ルータとルータの間にある部分（いわゆるセグメント）は、IPv6を有効にすればリンクローカルアドレスが動的に決まりますので、必ずしもIPv6アドレスを採番する必要はありません。
ですが、実際には遠隔からのping監視の必要性、インタフェースをアドレスで特定する必要性、などを考慮するとIPv6アドレスの採番が必要です。
ルータとルータの間の部分にIPv6アドレスを採番しつつ、ルータ自身に対してもロケータという形で/64のアドレスを割り当てます。
ロケータはルータの論理的なインタフェースと考えると分かりやすいと思います。

![fig_6_2](img/fig_6_2.drawio.png)

ルータのインタフェースおよびロケータにどういうアドレスを採番するか、はネットワーク設計者の腕の見せ所になります。
IPv6はアドレス空間が広大ですので、節約するよりも、効率的に集約することが重んじられます。
運用管理上意味のある単位でひとまとめにしたときに、そこで用いられている物理足のアドレスとロケータがまとめて一つのアドレスに集約できるように配慮して払い出します。

<br>

## ファンクション部

Function（ファンクション）は装置が自分自身の機能に対してセグメントIDを採番する対象です。

FunctionはRFC8986で標準化されていますので、ここからはその呼び方を使います。

> https://datatracker.ietf.org/doc/rfc8986/
>
> RFC8986 Segment Routing over IPv6 (SRv6) Network Programming

| ファンクション     |  説明  |
| ----------------- | ----- |
| End               | エンドポイント |
| End.X             | L3クロスコネクトのエンドポイント<br>Adjacency SIDと呼びます |
| End.T             | 特定のIPv6テーブルをルックアップするエンドポイント |
| End.DX6           | カプセル化を解除してIPv6クロスコネクトを行うエンドポイント<br>IPv4 L3VPN per-CE |
| End.DX4           | カプセル化を解除してIPv4クロスコネクトを行うエンドポイント<br>IPv4 L3VPN per-CE |
| End.DT6           | カプセル化を解除して特定のIPv6テーブルをルックアップするエンドポイント<br>IPv6 L3VPN per-VRF |
| End.DT4           | カプセル化を解除して特定のIPv4テーブルをルックアップするエンドポイント<br>IPv4 L3VPN per-VRF |
| End.DT46          | カプセル化を解除して特定のIPテーブルをルックアップするエンドポイント<br>L3VPN per-VRF |
| End.DX2           | カプセル化を解除してL2クロスコネクトを行うエンドポイント<br>L2VPN |
| End.DX2V          | カプセル化を解除してVLAN L2テーブルをルックアップするエンドポイント<br>EVPN Flexible Cross-connect |
| End.DT2U          | カプセル化を解除してユニキャストMAC L2テーブルをルックアップするエンドポイント<br>EVPN Bridging Unicast |
| End.DT2M          | カプセル化を解除してL2テーブルでフラッディングするエンドポイント<br>EVPN Bridging BUM |
| End.B6.Encaps     | カプセル化を伴うSRv6 Policyに紐付けられたエンドポイント<br>Binding SID |
| End.B6.Encaps.Red | End.B6.Encaps with reduced SRH |
| End.BM            | SR-MPLS Policyに紐付けられたエンドポイント |

たくさんありますが、よく目にするFunctionはこれらです。

<dl>
    <dt>End</dt>
        <dd>そのルータ自身への通信を処理する機能</dd>
    <dt>End.X</dt>
        <dd>ルータの自足のインタフェース上にいるノードに中継する機能</dd>
    <dt>End.DT4</dt>
        <dd>そのルータの中にあるL3VPNのIPv4ルーティングテーブル(VRF)を検索して中継する機能</dd>
    <dt>End.DT6</dt>
        <dd>そのルータの中にあるL3VPNのIPv6ルーティングテーブル(VRF)を検索して中継する機能</dd>
</dl>

<br>

## ファンクション部へのSIDの採番

人間が決めた値を静的に設定する方法と、プロトコルが動的に採番する方法があります。

静的に決める範囲は、上位1オクテットのうち 0x00 - 0x3f まで、合計64個としている実装が多いようです。
この場合、動的に決まるSIDは0x40以降になります。

<br>

### Endファンクション

Endファンクションはそのルータ自身宛ての通信を処理する機能ですから、そのルータを代表するSIDです。このSIDをノードSID(node-SID)と呼びます。

装置を代表するIPv6アドレスと、装置を代表するSIDは、関連付けておくとわかりやすくなります。
たとえばロケータが 2001:0db8:0:1::/64 であれば、ループバックにはその中から 2001:0db8:0:1::1/128 というIPv6アドレスを割り当てます。
ISISはSRv6のロケータ情報を/64の経路として配信しますので、ループバックに割り当てた/128のアドレスをconnected経路として再配送する必要はありません。

<br>

> **Note**
>
> SRv6を構成するルータのLoopbackのIPv6アドレスは {locator}::1/128 とする、といった具合にルール化しておくとよいと思います。

<br>

### End.X

End.Xは隣接ノードとの関係を示す機能です。このSIDを隣接関係SID(adj-SID)と呼びます。

隣接ノードを見つけるのはISISの仕事です。
ISISは隣接ノードを見つけ次第、ロケータの中からEnd.XのSIDを採番します。
この情報を詳細に調べると、そのインタフェースの先にいる隣接機器のIPv6アドレスを知ることができます。

動的に決めたEnd.XのSID(adj-SID)は装置の再起動やインタフェースのダウン・アップで変わってしまうかもしれません。
このインタフェースを通れ、と指定する場面では、事前にEnd.XのSIDが決まっていたほうがよいので、静的に決めて置いたほうがよいでしょう。

> **Note**
>
> EndとEnd.Xはどちらもそのノード自身のSIDです。Endは装置監視の宛先に、End.Xはインタフェース監視の宛先に使う、とざっくり考えていいと思います。

<br>

## End.DT4とEnd.DT6、End.DX4とEnd.DX6

End.DT4は自装置の中にあるVRF宛てのIPv4通信を処理する機能です。
End.DT6はVRF宛てのIPv6通信を処理する機能です。

ネットワークの設計によっては同じVRFの先に複数のCEルータがつながるケースがあります。
VRF単位にファンクションを割り当てるのがEnd.DT、CE装置単位にファンクションを割り当てるのがEnd.DXです。

<br>

> **Note**
>
> 装置の実装にper-vrfかper-ceかは異なりますので、マニュアルを参照しましょう。IOS-XRはper-ce、FITELnetはper-vrfがデフォルトの動作です。

<br>

> **Note**
>
> このVPNの通信はこの経路を通るようにしたい、といった特別な経路制御をかけたい場合はEnd.DTのSIDを静的に設定するのもありかもしれませんが、数が多くなることが予期されるので得策ではありません。
> VPNの通信に対して特別な扱いをしたい場合は、ISISのFlexAlgoを使うか、トラフィックエンジニアリングポリシーを適用するのが現実的です。

自装置のVRFに対して割り当てたSIDの情報は、iBGPを使って自分以外のエッジルータに配信します。
そのVRF宛ての通信は（iBGPのNext Hopアドレスではなく）配信したSID宛てになります。

<br>

> **Note**
>
> 実際にiBGPで交換している情報は、ロケータと **ラベル** の情報です。
> 昔からあるMPLS-VPNの仕組みをそのまま流用しているためです。
> 当然ですが、ラベルからSID、SIDからラベルに変換できます。

<br>

## SIDのルーティングテーブル

ネットワークを運用管理する立場からみると「あの機能にパケットを送るには、どのSIDを付ければよのか」ということを知りたくなります。
しかしながら、SRv6はアドレス体系であって、情報を交換するプロトコルではありませんので、SIDにルーティングテーブルというものは存在しません。
End.XはISIS、End.DTやEnd.DXはBGPの中に詳細情報が格納されています。
各プロトコルに関してのshowコマンドを駆使することになります。
現実的には全てのエッジルータからSIDの情報を集めて一元管理することになると思います。

> **Note**
>
> IOS-XRであれば、これらコマンドを駆使して経路とSIDの対応を調べます。
>
> show segment-routing srv6 sid
>
> show bgp vpnv4 unicast local-sids
>
> show bgp vpnv4 unicast received-sids
>
> show bgp vrf NAME local-sids
>
> show bgp vrf NAME received-sids
>
> show route vrf NAME PREFIX detail
>
> show cef vrf NAME PREFIX

<br>

## SIDに到達するための経路の確認

ISISやBGPの情報から宛先として使うべきSIDがわかったとして、そこに到達する経路はどうなるでしょう。

SRv6のSIDはIPv6と同じ形式ですから、IPv6のルーティングテーブルを検索すればSIDに到達するための経路が出てきます。
通常はロンゲストマッチのルールに従ってISISが配信した/64のロケータの情報にたどり着きます。

> **Note**
>
> 何らかの手段で/64のロケータの経路さえ学習すれば、途中にSRv6を理解しないルータがいても大丈夫です。

<br>

## L3VPN通信のパケット

はじめに、比較のため広く用いられているMPLS VPNの場合を考えます。
L3VPNはPEルータ同士がiBGPを使ってアドレスファミリVPNv4の情報を交換することで成立します。
VPNv4の経路情報は、あのPEの中にある、あのVRFの先にいる経路、という情報です。

![fig_7.mpls_vpn](img/fig_7.mpls_vpn.drawio.png)

MPLS網の中に流れるパケットの宛先は、PEルータを指します。具体的にはPEルータのiBGPのピアのアドレスに相当するMPLSラベルです。
iBGPでは装置を代表するアドレス、すなわちループバックのアドレスでピアリングするのが普通です。
そのアドレスにたどり着くためのラベルが一段目のラベルです。二段目のラベルには、どのVRFの通信か、を識別する情報が入ります。

このパケットをMPLS網のコアルータが観察しても、なんの通信か、を察することはできません。
二段目のラベルの意味を知っているのはiBGPでVPNv4の情報を交換したPEルータだけだからです。

たとえば、契約にもとづいてVPNの通信に帯域制御をかける、という要件があったとします。
MPLS VPNでこの要件を実現するのはなかなか難しく、PEルータがパケットを受信した時点で帯域を制御しなければいけません。

MPLS VPNのポイントは以下の通りです。

- ラベルを2段使う
- 一段目は出口になるPEルータのループバックアドレスに向けたラベル
- 二段目はVRFを識別するラベル
- コアルータが一段目のラベルを見ても、出口になるPEルータ宛ての通信にしか見えない（VPNの通信かどうかもわからない）
- コアルータが二段目のラベルを見ても、その意味は分からない（意味を知っているのはiBGPで情報を交換したPEルータだけ）

<br>

今度はSRv6の場合を考えてみます。

PE間でiBGPを使って情報を交換する仕組みはMPLS VPNの場合と同じです。
End.DT4では、VRFごとにSIDを採番します（CE側に経路情報が複数あっても、同じVRFに収容される限り同じSIDが採番されます）。
iBGPでアドレスファミリVPNv4を交換する際、SIDの情報がラベルに変換されてiBGPで伝搬し、もう一度ラベルからSIDに逆変換されます。

![fig_8.srv6_vpn](img/fig_8.srv6_vpn.drawio.png)

実際のパケットは、End.DT4のSIDが宛先になります。iBGPのNext Hopではありません。SIDです。

これがSRv6の特徴の一つです。
MPLS VPNのときには一段目のラベルとして出口になるPE（より正確には出口PEのループバックアドレス）が使われていました。
出口になるPEにVRFが複数あっても、iBGPのNext Hopが出口になりますので、一段目のラベルは同じものが使われます。

一方、SRv6 VPNではVRFに相当するSIDが宛先になります。
SIDはIPv6アドレスそのものです。
ロケータの中から割り当てられますから、どのルータが所持しているものなのかもわかります。
コアルータはSIDの意味まではわかりませんが、ロケータの/64の経路を見ればどこに中継すればいいかわかります。

SRv6 VPNのポイントは以下の通りです。

- IPinIPでトンネルする
- 外側ヘッダの宛先はVRFに割り当てられたSIDであり、出口ルータのループバックではない
- 途中経路を指定しなければSRヘッダは付与されず、単なるIPinIPパケットで運ばれる

VRFが決まれば、End.DT4のSIDが決まり、網内で中継されるパケットの宛先IPv6アドレスが決まります。
契約にもとづいてVPNの通信に帯域制御をかける、という要件があったとしても、単純に宛先IPv6アドレスで帯域制御するだけです。
この特徴はSRv6を使うことで得られるメリットの一つです。

<br>

## ヘッドエンド側の動作

基本はIPinIPです。

SRv6ネットワークをファブリックと考えて、パケットが着信した入り口ルータでは元のパケットをまるごと別のIPv6パケットで包み込み、出口になるSIDを目掛けて中継します。
出口ルータはSIDの機能に応じた処理を行います。L3VPNのSID宛てなら、VRFのテーブルをみて適切にVPNに転送します。

> **Note**
>
> micro-SIDはIPinIPではなく、これとはまったく異なる動きをします。

<br>

## コアルータの動作

宛先アドレスが自分宛てでなければ、単なるIPv6パケットとして中継します。

自分を目掛けて飛んできたIPv6パケットの宛先アドレスがEndやEnd.XのSIDであれば、SRv6としての処理を行います。

<br>

## パケットにSRヘッダが付く場合の通信

どこを経由してたどり着け、と指示する場合にはパケットにSRヘッダが付きます。

- RFC 8754 IPv6 Segment Routing Header (SRH)

https://datatracker.ietf.org/doc/rfc8754/

```text
   The SRH is defined as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Next Header   |  Hdr Ext Len  | Routing Type  | Segments Left |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  Last Entry   |     Flags     |              Tag              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |            Segment List[0] (128-bit IPv6 address)             |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                                                               |
                                  ...
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |            Segment List[n] (128-bit IPv6 address)             |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    //                                                             //
    //         Optional Type Length Value objects (variable)       //
    //                                                             //
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Segment Listは配列です。SIDすなわちIPv6アドレスと同じ形式のものが並びます。

外側のIPv6ヘッダの宛先アドレスはホップバイホップで変わっていきます。

TODO: 絵と説明を入れる


<br>

## SIDにpingを打ちこむと？

EndとEnd.XのSIDであれば、応答が返ってきます。

それ以外のSIDは形式こそIPv6と同じですが、特別な機能に対して割り当てた番号ですので、そこにpingを打ち込んでも応答は期待できません。

<br>

## 狙った経路で通信する

ルータの中に存在する機能にSIDを採番したとして、そこに向けて通信するにはどうしたらいいかを考えます。

一番簡単な方法はスタティックルーティングです。
「あのSIDにたどり着くためには、ここを通過せよ」というポリシー情報をルータに設定します。
ただし、これは片道なので戻りの通信も同様にスタティックルートが必要になります。

<br>

FlexAlgoを使ってISISの経路計算を論理的に分割する方法もあります。
OSPFのコスト、ISISのメトリックで経路を計算すると最短パスが求まるわけですが、回線の遅延時間で計算すると違う経路の方が優れているかもしれません。
経路計算に使う指標を複数導入して、どの指標で計算した経路を選択するか制御するのがFlexAlgoです。

FlexAlgoではインタフェースに特別な名前をつけて、この名前のインタフェースは通るな、といった制約を課して経路計算することもできます。

![FlexAlgo](img/flexalgo.drawio.png)

SRv6で用いられるIGPが事実上ISISの一択になっているのは、こういった特別な経路計算を得意としているからです。

<br>

## L3VPNの通信

エッジルータがVPN用にVRFを作成すると、VRFごとにSIDが割り当てられます（End.DT4 per-VRFの場合）。

このSIDはBGPで自分以外の全てのエッジルータに配信されます。

エッジルータはそのSIDに向けてIP-in-IPでカプセル化してVPNのパケットを送信します。

<br>

> **Note**
>
> SRv6を使っていない場合は、iBGPで経路を配信したエッジルータのアドレスがnext-hopになります。
> SRv6ではnext-hopがSIDに置き換わります。

ロケータの情報はISISで配られていますので、その経路に従ってIP-in-IPパケットがネットワークに流れていきます。

この時点ではまだSRv6のヘッダは登場しません。
途中経路上でパケットをキャプチャしても単なるIP-in-IPのパケットしか観察できません。
特別な経路を通って到達させたいときに初めてSRv6が使われます。

「あのSIDにたどり着くためには、ここを通過せよ」という情報をルータに設定するのは、
スタティックルーティングのときとまったく同じですが、
VPNの場合には、より柔軟なポリシーを作れるようになっています。

1. BGPで学習したNext-HopとなるSIDであり、かつ
2. BGPのColor属性が一致した場合、
3. この経路を使用せよ、

というポリシーをエッジルータに作成しておきます。
BGPの経路にカラー属性を付与することで、通る場所を制御できるようになります。

<br>

> **Note**
>
> BGPで経路を配信する側が、通り道を制御する、ということです。

何かしら特別な契約がある場合はカラー属性を付与して特別な経路を通過、そうでなければベストエフォートの経路を通過、というようなユースケースに使えそうです。

<br>

## PSP = Penultimate Segment Pop

SIDのテーブルを表示するとPSPマークが出てきます。最終目的地の手前でSRヘッダを取り除くことを言います。
不要になったSRヘッダはなるべく早めに取り除いたほうが、転送効率の観点で好ましいです。

隣接ルータがSRv6を構成していない場合、最終目的地の手前であってもSRヘッダを取り除く必要があります。
そのような場面以外は、あまり気にしなくてよいと思います。

> **Note**
>
> SR-MPLSでVPN通信を運ぶ場合、運んでいるIPパケットのToSの値はラベルのExpフィールドに反映されますので、PEの手前でラベルを外してしまうとその情報が失われてしまいます。
> そのためPEルータではexplicit-nullを設定してNULLラベルを意図的に使うこともありますが、SRv6では特にそのようなことは気にしなくてよいと思います。

<br>

## uSIDの利用

マイクロSID(Micro-SID)をuSIDと表記します。uはマイクロの記号μの代用です。

フルレングスのSIDひとつの中に、コンパクトなSIDを複数格納することで効率とスケーラビリティの向上が期待できます。

uSIDは
`<uSID-Block><Active-uSID><Next-uSID>...<Last-uSID><End-of-Carrier>...<End-of-Carrier>`
の形式をとります。
uSID-Blockは共通部のIPv6プレフィクスで、続いてuSIDのIDの配列が続きます。
End-of-CarrierはuSIDの終わりを表現する定義済みのuSIDで`0000`です。
uSIDが並んだ後の余りの部分は0でパディングされる、と考えていいと思います。

uSIDの形式は `F bb uu` の形式で表現され、F3216と表記した場合はuSID-Blockの長さが32ビット、uSID IDの長さが16ビット、合計48ビット、という意味になります。

> **Note**
>
> IOS XR 7.3で実装されているuSIDはF3216形式のみです。

uSIDに関連付けられたエンドポイントの動作は、それ用に定義されています。

- uN : End動作のNEXT-CSID (圧縮された SID) 表記
- uA : End.X 動作のNEXT-CSID表記
- uDT : End.DT 動作のNEXT-CSID表記
- uDX : End.DX 動作のNEXT-CSID表記

具体的な例で考えてみます。F3216を想定すると、ISPから入手する/48のアドレスは使えませんので、RFC4193(Unique Local IPv6 Unicast Addresses)のローカルアドレスを利用します。
RFC4193に従うと0xFDに続けてランダムな数字を生成するべきですが、ここでは見やすいように先頭32ビットをFD00:0000:とします。

> **Mote**
>
> uSIDではRFC4193 ユニークローカルアドレスの利用が推奨されます。

![micro-SID](img/usid.drawio.png)

各ノードには16ビットのuNが割り当てられます。PE03のvrfにはuDT4で`0303`が割り当てられ、PE05のvrfには`0505`が割り当てられたとします。

PE03がPE05のvrfにたどり着く経路として PE03-CR01-CR02-CR01-PE05 を指定すると、宛先になるIPv6アドレスは `fd00:0000:0100:0200:0100:0500:0505:0000` となります。

- ①のFD00:0000はuSID Blockです。
- ②の0100はCR01のuNです。
- ③の0200はCR02のuNです。
- ④の0100はCR01のuNです。
- ⑤の0500はPE05のuNです。
- ⑥の0505はPE05のvrfのuDTです。

このアドレスの経路情報を探索すると、ロンゲストマッチのルールで先頭48ビット `fd00:0000:0100` の経路情報に一致します。
これはCR01がISISで配信しているはずのものです。したがってPE03から送信されたパケットはまずCR01に向かいます。
CR01は自身のuNの次に来るuNを見て次に転送すべきノードを0200、すなわちCR02と特定します。
宛先アドレスから自身のuNを取り除き、新たな宛先アドレスとして `fd00:0000:0200:0100:0500:0505:0000:0000` を組み立てて転送します。こうして最終目的地のPE05までたどり着くと、PE05のuNである0500の次にくる0505が自身のuDTであることを特定し、vrfに中継します。

このようにuSIDを使うとSRv6ヘッダを使うことなく、宛先IPv6アドレスだけで経由地を複数指定できます。

> **Note**
>
> F3216形式のuSIDであれば一つの宛先に5個まで経由地を指定できます。










<br>

##

<br>

## 疎通確認の方法

IOS-XRの場合は、ポリシー名を指定してping、tracerouteを打てます。

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

ただ、現場作業時にNSOを持ち込むのはオーバースペックなので、簡易なツールも必要。
pyATSを使って各種操作を自動化するとよい。

作成済みスクリプトの例
- ファブリック内のSIDを全て収集
- BGP状態の確認
- ISIS状態の確認
- パラメータシートからのVPNプロビジョニング
- end-to-endの疎通確認

<br><br><br><br>

# RFC

RFC8986は必読です。

## アーキテクチャ関連

RFC 8402 Segment Routing Architecture

RFC 7855 Source Packet Routing in Networking (SPRING) Problem Statement and Requirements

RFC 8660 Segment Routing with MPLS data plane

RFC 8754 IPv6 Segment Routing Header (SRH)

RFC 8986 Segment Routing over IPv6 (SRv6) Network Programming

## ISIS関連

RFC 7810 IS-IS Traffic Engineering (TE) Metric Extensions

RFC 8491 Signaling MSD (Maximum SID Depth) using IS-IS

RFC 8667 IS-IS Extensions for Segment Routing

RFC 8668 Advertising L2 Bundle Member Link Attributes in IS-IS

## BGP関連

RFC 8571 BGP-LS Advertisement of IGP Traffic Engineering Performance Metric Extensions

RFC 8669 Segment Routing Prefix SID extensions for BGP

## OSPF関連

RFC 7471 OSPF Traffic Engineering (TE) Metric Extensions

RFC 8665 OSPF Extensions for Segment Routing

RFC 8666 OSPFv3 Extensions for Segment Routing

RFC 8476 Signaling MSD (Maximum SID Depth) using OSPF

## OAM関連

RFC 8287 Label Switched Path (LSP) Ping/Trace for Segment Routing Networks Using MPLS Dataplane

RFC 8403 A Scalable and Topology-Aware MPLS Dataplane Monitoring System
