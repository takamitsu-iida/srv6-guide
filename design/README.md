# ネットワーク設計

設計しないといけないことをメモ。

<br><br>

## SRv6固有の考慮事項

SRv6といってもバリエーションがあり、何を実現したいかで選択すべき設計が変わってきますし、機器の実装状況によっても設計を左右します。

まず大きいのは、ポリシーを使ったトラフィックエンジニアリングをするかどうか、です。
この場合はIOS-XRでの実装状況を鑑みてフルレングスではなくuSIDを使った方が良さそうです。
すると、uSIDを実装していない機器が排除されてしまいます。

ポリシーを使ったトラフィックエンジニアリングを実戦投入するのは時期尚早な印象もありますので、そこまでは不要、ISISでFlexAlgoができればよい、という場合もあるかと思います。
この場合はフルレングスのSIDでいいでしょう。マルチベンダで構成できると思います。

もちろん、フルレングスのSIDとuSIDは排他ではありませんので、同時に併用する、という選択もありえます。

uSIDを利用するときには、ユニークローカルアドレスの利用が推奨されます。
ベースになるIPv6ネットワークもuSIDにあわせてユニークローカルアドレスで統一するか、
ベースのネットワークはグローバルIPv6アドレスでuSIDのみローカルアドレスにするか、これも悩ましいところです。

まずはフルレングスのSRv6を構成して、uSIDをアドオンする、というのが無難な選択肢なのかもしれません。


<br><br>

## 物理ネットワーク

実はこれが一番難しいのではないかと思っています。

FlexAlgoやTEポリシーを導入すると、ネットワークを論理的に分割するような使い方ができます。
いわゆるスライシングと呼ばれているものです。

パケットが物理的に同じ場所を通っていたらスライシングしても意味がありませんので、提供するサービスレベルと物理ネットワークをどう対応づけて設計するか、という難しい問題になります。


<br><br>

## ISISネットワーク

SRv6では動的ルーティングプロトコルにISISを使うのが事実上の標準です。
ISISはOSPFと同様、リンクステート型のルーティングプロトコルですが、ネットワーク設計の観点でOSPFと異なる点があります。

OSPFではインタフェースに対してどのエリアに属するかを指定しますので、エリアの境界線がルータの中にあります。
ISISはルータがどのエリアに属するか、を指定します。したがってエリアの境界線はリンク上にあります。
レベル2はバックボーンエリアに相当しますが、エリア番号は0である必要はありません。

L2ネットワーク一つで全域をカバーするか、エリアに分割して経路集約をかけていくかは、
ネットワークの規模やWANの有無、管理する範囲、など様々な要因を考慮して決めていきます。

具体的なイメージを思い描くために、データセンター同士をWAN回線で接続するケースを想定してみます。
Central-DCがバックボーンエリアに相当しますのでルータは全てL2になります。
East-DCとWest-DCのエッジルータはバックボーンエリアと接しますのでL1/2ルータとなり、
経路を集約してCentral-DC側に広告します。

East-DCとWest-DCのその他のルータはデフォルトルートをL1/2ルータに向ければよいので、L1ルータとなります。

![ISISネットワーク](img/isis.drawio.png)

<br><br>

## IPv6ネットワーク

装置に割り当てるロケータは論理的なインタフェースであり、そこに/64のIPv6アドレスを割り当てる、と考えると分かりやすいです。
SRv6ネットワークはアドレス設計の観点でいうと単なるIPv6のネットワークにすぎません。

> **Note**
>
> RFC4193 Unique Local IPv6 Unicast Addresses
>
> を使ってもいいかもしれません。
> RFC4193ではfd00::/8に40ビットのランダムな文字列を生成してfdxx:xxxx:xxxx::/48として利用します。ランダムな文字列を生成するツールは多数あります。
>
> https://network00.com/NetworkTools/IPv6LocalAddressRangeGenerator/


ISP経由で取得できるIPv6アドレスは最大で/48のサイズになりますので、ここでは2001:0db8:0000/48を例に考えます。
/48を分割して利用しますが、このアドレス全てをインフラに使えるわけではありませんので、細かく分割していきます。
分割の単位は/52、/56、/60にするとアドレス表記の切れ目と一致してわかりやすくなります。

![IPv6アドレス設計](img/address.drawio.png)

2001:0db8:0000/48のアドレスを/52で分割すると、2001:0db8:0000:**0**0/52から2001:0db8:0000:**F**0/52まで合計16個に分割されます。
仮に、このうちの先頭1個、2001:0db8:0000:**0**0/52をインフラ用に割り当てるとします。
これをさらに/56で分割して、16個の/56を取り出します。Center-DCには/56を4個分、East-DCには2個分、West-DCには2個分、といった具合に割り当てていきます。

<br>

> **Note**
>
> IPv6のアドレス表記の文字一つは4ビット(0～F)なので、/48から4ビット単位に/52, /56, /60の単位で考えるとわかりやすいです。


<br><br>

## ロケータ設計

ロケータのブロック部は、SRv6を構成するルータで共通でなければいけません。所有しているアドレスが/48であればその条件は最初から満たしています。
RFC4193のユニークローカルアドレス(fc00::/7)を使う場合には、先頭40ビットが装置によって異なるようなアドレス割り当てをしないように気をつけます。
将来的にFlexAlgoを導入することを考慮すると、ロケータ（論理インタフェース）はルータ1台あたり複数必要になることを想定しておくべきです。
Affinityのカラーに8色使うなら、それだけでルータ1台に3ビット分のブロックが必要です。

> **Note**
>
> IOS-XRのFlexAlgoでサポートするロケータの数は8個です。
> このことから各ルータに最大8個のロケータを割り当てることを想定しておけばよいと思います。

たとえば、Central-DCに4つの/56を割り当てたとします。

- 2001:0db8:0000:00/56
- 2001:0db8:0000:01/56
- 2001:0db8:0000:02/56
- 2001:0db8:0000:03/56

このうち、先頭の2001:0db8:0000:00/56からロケータを取り出すと、各ルータに割り当てるロケータはこのようになります。

![ロケータ設計](img/locator.drawio.png)

ここではロケータにaという名前を付けています。
FlexAlgoを使うとaffinityごとにロケータを持つことになりますので、a面、b面であったり、gold、silverのような色の名前をつけてもよいでしょう。

<br><br>

## ループバックアドレス

ループバックは装置を代表するアドレスです。
採番したロケータの中からIPv6アドレスを割り当てるのが最善です。

```
{locator}::1/128
```

のようなルールで設計するとよいでしょう。

<br><br>

## ISISのインタフェースのネットワークタイプ

ルータ間の接続はPoint-to-Pointにします。

トラフィックエンジニアリングはp2pであることが前提条件になります。

<br><br>

## ルートリフレクタとISISメトリック設計

PEルータでVPNを作るときには、PEルータ間でiBGPを使った情報交換をします。
iBGPはフルメッシュで構成しますが、実際にフルメッシュでピアリングするのは困難ですので、ルートリフレクタを構成します。
ルートリフレクタはクラスタを組んで冗長化します。

![Route Reflector](img/rr.drawio.png)

各PEルータからRRにたどり着くための経路がどうなっているかは、把握しておいた方がよいでしょう。
PE-RR間でBFDを使って迅速に障害検知する場合、シングルポイントの障害で両系のiBGPピアがダウンしかねません。

BFDを使わない、という設計も選択肢の一つです。

<br><br>

## MTU/MSS

MTUはネットワーク内で統一します。SRヘッダが付くことを考慮してMTUを十分大きくしておきます。

（FITELnetは物理足に設定するわけではないので、注意が必要）

VPN通信が通るトンネルのインタフェースはTCP MSSの書き換えをする、しないを選択します。

<br><br>

## ToSの伝搬

L3VPNを中継する際、アンダーレイのIPヘッダにToSを反映させるか、させないか、を設計します。

反映させない場合、エンドユーザが付けたToSがend-endで透過しなくなります。


<br><br>

## スタティックTEの途中経路指定

TE（トラフィックエンジニアリング）は一方通行のトンネルですので、そのトンネルで相手まで通信できるか、プロトコルで確認していません。
途中経路として「ここを通れ」というポリシーを作る際、そこにたどり着く経路は動的ルーティングで冗長化されていないと、シングルポイント障害で通信が止まってしまいます。

<br><br>

## オーケストレータ

SRv6はファブリックネットワークですので、トラフィックの入り口と出口を強く意識します。
何か設定を施したり、VPNをプロビジョニングしたり、情報を収集したりする際に、複数の装置を同時に扱う必要がでてきます。

何かしらツールがないと運用で困りそうです。

CiscoであればNSO（Network Services Orchestrator 旧Tail-F社）という製品が用いられることが多いようです。

<br><br>

## PCE

iBGPにおけるルートリフレクタのようなイメージで経路計算を集約できるようになります。ポリシーを適用する際にはPCEを導入した方が良いでしょう。

<br><br><br><br>

# 設計項目

基本的な設計項目をリスト化しておきます。

- 基盤ネットワーク設計
    - 物理ネットワーク
        - DC内ネットワーク
            - 面設計
                - 1系-2系 or A面-B面
                - 面のスケールアウト方式
            - トポロジ
                - ラダー型 or ツリー型
                - ルータ種別、命名規則
                    - CR or P = コアルータ
                    - ER or PE = エッジルータ
                    - WR = WANルータ
                    - CE = カスタマーエッジルータ
                - コア部
                    - CRルータ間接続
                - エッジ部
                    - PE-CR間接続
                    - PE-CE間接続
                        - L3VPNシングルセグメント
                            - VRRP or Anycast Gateway
                        - L3VPNマルチセグメント
                            - eBGP
                        - EVPN
                            - ESI LAG
                        - L3VPN + EVPN同時利用

            - 物理インタフェース
                - 速度
                - ファイバ種別、コネクタ形状
                - MTU

        - DC間接続
            - 回線冗長方式
                - 現用・待機
            - 接続プロトコル
                - ISIS or BGP

    - アンダーレイプロトコル
        - IPv6
            - アドレス設計
            - Loopbackインタフェース
            - インタフェースアドレス採番規則

        - BFD
            - Hello
            - BFD乗数

        - ISIS
            - プロセス名
            - エリア番号
            - ルータタイプ
                L1/L2/L1-2
            - インタフェースタイプ
                - point-to-pointのみ
            - メトリック設計
            - Hello/Dead
            - エリア設計
            - アドレス集約
            - 装置起動時のプロセス待機（max-metric router-lsa on-startup [time]）
            - BFD対応
            - NSF対応

        - SRv6
            - ロケータ設計
                - 用途ごとのロケータ
                - uSID利用
            - encap sourceアドレス
            - logging
                - SID status
            - PSP（EVPN時に制約あり？）
            - 静的SID割り当て
                - node-SID/ajd-SID
                - End.DT
            - FlexAlgo
                - TE metric
                - affinity
            - SR-TE
                - PCE
                - BGPオンデマンドポリシー

        - iBGP
            - AS番号
            - RR
                - クラスタID
            - keepalive/hold
            - アトリビュート
                - MED
                - AS PATH (prepend)
                - Local Pref
            - nexthop tracking(bgp nexthop trigger delay)
                - critical
                - non-critical

    - オーバーレイプロトコル

        - L3VPN
            - VRF
                - 名前
                - RD/RT

            - address-family vpnv4
                - SRv6ロケータ

            - address-family vpnv6
                - SRv6ロケータ

            - address-family ipv4 vrf/ipv6 vrf
                - AS番号
                - keepalive/hold timer
                - BFD有無
                - アトリビュート
                    - local pref(1系2系)
                    - med(なし)
                    - as-path prepend
                - 経路再配送
                - 経路数制限

            - シングルセグメント構成
                - anycast gateway address
                    - IP/MAC

            - MTU
            - ToS/DSCP反映有無
            - TCP MSS変換有無

        - EVPN
            - Bridge Group
            - Bridge Domain
                - 文字列 L2VPN単位
            - EVI番号
                - 1-65534 L2VPN単位
                - RD/RT
            - ESI
                - 16進数 物理ポート単位
            - storm control
                - broadcast/multicast/unknown-unicast
            - ESI-LAG
                - Bundle-Ether番号(Port-channel)番号
                - LACP
                    - system mac
                    - period(short/long)
            - isolation group
                - 適用有無
            - NextHop選択（lowest ip/highest ip/modulo）
            - BUM
                - split holizon
            - VLAN-ID書き換え
            - L2制御フレーム(BPDU/LLDP/CDP...)
                - フィルタリング or 透過
            - 装置起動時のプロセス待機（startup-cost-in [time]）

    - 信頼性設計
        - リンク障害時の通過経路
        - ネットワークの面冗長設計
            - Active-Standby or Active-Active
        - SRv6 microloop avoidance
        - SRv6 TI-LFA
            - node protection
            - link protection
        - L3VPN収容
            - eBGP
                - ピア構成
            - ESI-LAG併用時
                - anycast gateway or vrrp or hsrp
        - EVPN収容
            - ESI-LAG
                - Active-Active or Active-Standby(port-active設定)
        - サイレント障害検知
            - SR DPM(Data Plane Monitoring)のSRv6版はある？

    - ネットワーク性能設計
        - 帯域制御
            - 適用箇所
            - 方式
                - 契約種別
                    - ベストエフォート
                        - 共有ユーザ数
                    - 帯域確保
                - classify粒度
                    - 管理系通信(BGP/ISIS/BFD/ICMP/...)
                    - vlan/ip/tcp
                - shaping/policing
        - RTT計測

    - 拡張設計
        - 収容数増加時のスケールアウト単位
        - 装置一台あたりの許容スペックの明確化
            - 論理インタフェース数
            - Bridge Domain数
            - bundle-ether数
            - EVI数
            - BVI数
            - VRF数
            - IP経路数
            - MAC数
        - WAN回線不足時
            - スケールアップ
            - スケールアウト

    - ネットワークセキュリティ
        - BGP経路数保護
        - L2ループ対策
            - ストーム制御
        - 装置管理接続
            - アドレス制限
            - 認証
        - CPU保護
            - CoPP
        - 証跡管理
        - 装置OSの脆弱性対応

- 運用保守
    - in-band or out-band
    - プロビジョニング手段
        - オーケストレータ
    - SNMP
        - バージョン、認証、trap、community
    - gRPC
    - NTP
        - サーバ
    - 監視
        - ICMP監視
        - SNMP Trap
        - 障害監視
            - 装置
            - リンク
            - 環境（ファン・電源）
        - トラフィック量監視
            - 回線使用率
            - エラーパケット数
        - フロー監視
            - NetFlow
        - キャパシティ監視
            - CPU
            - メモリ
    - ログ
        - SYSLOG
        - gRPC or NETCONF

    - ライフサイクル管理
        - ソフトウェアバージョンアップ
    - メンテナンス停止
    - 保守交換手順
