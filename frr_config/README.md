# Ubuntu Server 22.04 L3VPN over SRv6

LinuxだけでL3VPNを構成します。

[SRv6 BGP](README.bgp.md) IPv4 over SRv6構成の設定例です。BGPを使ってVPNの経路情報を交換します。

[SRv6 STATIC](README.static.md) L3VPN over SRv6構成の設定例です。BGPを使わずにスタティックにSIDを設定します。

<br><br>

## 留意事項

- Ubuntu Server 22.04を使います

- 動的ルーティングのデーモンとしてFRRを使います
