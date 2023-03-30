# SRv6関連情報まとめ

IETFでSRv6の標準化が始まったのは2014年です。
その後Ciscoが製品に実装して世に出したのが2019年12月。
SoftbankやChina Telecom等、大規模な商用ネットワークでの稼働実績もあり、安心して導入できる環境が整ってきています[【１】][draft-matsushima-spring-srv6-deployment-status-15.txt]。

[draft-matsushima-spring-srv6-deployment-status-15.txt]: https://www.ietf.org/archive/id/draft-matsushima-spring-srv6-deployment-status-15.txt

SRv6の応用範囲は広いにもかかわらず一般的な企業ネットワークでの稼働実績はほとんどないのですが、だから知らなくて良い、というわけではありません。

直接目に触れることのない部分でVXLANやMPLSが稼働しているインフラは多数あります。
それらの一部は今後SRv6に移行していくでしょう。
クラウド周辺やコンテナ周辺がSRv6をサポートするかもしれませんし、SD-WANを構成する機能要素としてSRv6が必要になるかもしれません。

ネットワーク分野のエンジニアであれば知識としてしっかりと身につけておいたほうがよいと思います。

<br><br>

# [SRv6とは](doc/README.md)

<br><br>

# [SRv6ネットワーク設計](design/README.md)

<br><br>

# [IOS-XR設定例](iosxr_config/README.md)

<br><br>

# [Linux FRR設定例](frr_config/README.md)
