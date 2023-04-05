# Ubuntu Server 22.04 L3VPN over SRv6

LinuxだけでL3VPNを構成します。

[SRv6 BGP](README.bgp.md) IPv4 over SRv6構成の設定例です。BGPを使ってVPNの経路情報を交換します。

[SRv6 STATIC](README.static.md) L3VPN over SRv6構成の設定例です。BGPを使わずにスタティックにSIDを設定します。

[srext](README.srext.md) カーネルモジュールsrextを使ってproxy動作を試しましたが、動作しなかった例です。参考までに記録を残します。

[SRv6 SFC](README.srext.md) LinuxでSFC(Service Function Chaining)を構成する設定例です。

<br><br>

## 留意事項

- Ubuntu Server 22.04を使います

- 動的ルーティングのデーモンとしてFRRoutingを使います
