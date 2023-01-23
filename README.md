# SRv6関連情報まとめ

IETFでのSRv6の標準化は2014年に始まったそうです。
その後Ciscoが製品に実装して世に出したのが2019年12月です。

SoftbankやChina Telecom等、大規模な商用ネットワークでの稼働実績もあり、安心して導入できる環境が整っていると思います[【１】][draft-matsushima-spring-srv6-deployment-status-15.txt]。

[draft-matsushima-spring-srv6-deployment-status-15.txt]: https://www.ietf.org/archive/id/draft-matsushima-spring-srv6-deployment-status-15.txt

一方で、SRv6の応用範囲は広いにもかかわらず、一般的な企業ネットワークでの稼働実績はほとんどありません。
だから知らなくて良い、というわけではありません。

直接目に触れることのない部分でVXLANやMPLSで稼働しているインフラは多数あります。
それらの一部はSRv6に移行していくでしょう。
クラウド周辺やコンテナ周辺がSRv6をサポートするかもしれませんし、SD-WANを構成する機能要素として必要になるかもしれません。

ネットワーク分野のエンジニアであれば知識としてしっかりと身につけておいたほうがよいと思います。

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

<br><br>

[SRv6とは](doc/README.md)

[SRv6ネットワーク設計](design/README.md)

[IOS-XR設定例](iosxr_config/README.md)

<!--
広域のSRv6で活用場面はないか？

各拠点にインターネット回線を引き込んでSRv6を構成すると、
SRv6 --- (インターネット) --- SRv6
という構成ができる。

このとき何かいいことできないかな？
ロケータにグローバルIPv6を使うと、インターネットのどこからでもSID目掛けて通信できるわけで、この特性を使って何かできないかしら？
クラウド側にロケータが配置されたとして、何かできないかしら？

-->
