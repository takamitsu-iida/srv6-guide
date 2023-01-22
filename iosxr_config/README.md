# IOS-XR SRv6 設定例

以下の順番で設定を追加していきます。

[SRv6 BGP](README.srv6_bgp.md) IPv4 over SRv6構成の設定例です。

[SRv6 L3VPN](README.srv6_l3vpn.md) L3VPN over SRv6構成の設定例です。

[SRv6 L3VPN uSID](README.srv6_l3vpn_usid.md) L3VPN over SRv6 uSID構成の設定例です。

[SRv6 L3VPN uSID FlexAlgo](README.srv6_l3vpn_usid_flexalgo.md) L3VPN over SRv6 uSID FlexAlgo構成の設定例です。

<br><br>

## 留意事項

- XRv9000はSRv6を実装しています

- XRv9000はSRv6 uSIDを実装しています

- XRv9000はSRv6 TEを実装しています

- XRv9000でTE Policyを使うためにはuSIDが必要です

- CRS1000vはEVPNを実装しています

- CSR1000vはSRv6を実装していません

<br><br>

## 検証環境

IOS XR Version 7.7.1

<br><br>

## 参照したマニュアル

https://content.cisco.com/chapter.sjs?uri=/searchable/chapter/content/en/us/td/docs/iosxr/ncs560/segment-routing/73x/b-segment-routing-cg-73x-ncs560/m-configuring-srv6-traffic-engineering-ncs5xx.html.xml#id_125396

https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/segment-routing/78x/b-segment-routing-cg-ncs5500-78x/configure-srv6-micro-sid.html
