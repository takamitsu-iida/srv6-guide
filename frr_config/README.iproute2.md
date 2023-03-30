# SRv6 Linux設定

> 参考
>
> https://segment-routing.org/


実機では試してません。

メモです。

### インタフェースごとの設定

```
net.ipv6.conf.*.seg6_enabled (integer)
```

0はSRパケットを破棄

1はSRパケットを受信して処理


### カプセル化

```
ip -6 route add <prefix> encap seg6 mode <encapmode> segs <segments> [hmac <keyid>] dev <device>
```

- prefix: 宛先プレフィクス
- encapmode: encap もしくは inline
- segments: コンマ区切りのSID 例： fc00::1,fc42::5
- keyid: HMAC key ID
- device: ループバック以外のデバイス

### ソースアドレスの指定

```
ip sr tunsrc set <addr>
```

デフォルトではインタフェースのアドレスが選ばれる。

リンクローカルアドレスしかなければ、loが選ばれる。


### SIDテーブル

```
# echo 100 localsid >> /etc/iproute2/rt_tables
# ip -6 rule add to fc00::/64 lookup localsid
# ip -6 route add blackhole default table localsid
```

table 100をlocalsidとして作成。

fc00::/64は作成したlocalsidを参照。

その他の知らない宛先は破棄。

/etc/iproute2/rt_tablesには以下の通り、既定で設定されている。

```
root@pe3:~# cat /etc/iproute2/rt_tables
#
# reserved values
#
255     local
254     main
253     default
0       unspec
#
# local
#
#1      inr.ruhep
```

### SIDとファンクションの対応付け

```
ip -6 route add <segment> encap seg6local action <action> <params> dev <device> table localsid
```

- segment: SIDを指定。プレフィクスでもよい
- action: ファンクション
- params: そのファンクションに与えるパラメータ
- device: ループバック以外のデバイス

ファンクションに指定できるのは以下の通り。

- End: 通常のSRヘッダ処理

- End.X nh6 <nexthop>: 通常のSHヘッダ処理の後、指定されたネクストホップにパケットを転送

- End.T table <table>: 通常のSRヘッダ処理の後、指定したルーティングテーブルを参照してネクストホップに転送

- End.DX2 oif <interface>: L2フレームのカプセル化をといて、指定したインタフェースに転送

- End.DX6 nh6 <nexthop>: このノードが最後のSIDである、もしくはSRヘッダがない場合、インナーIPv6パケットのカプセル化をといて、指定したネクストホップに転送

- End.DX4 nh4 <nexthop>: DX6と同じ動きをインナーIPv4パケットで実施

- End.DT6 table <table>: decapsulate an IPv6 packet and forward it to the next-hop looked up in the specified routing table.

- End.B6 srh segs <segments> [hmac <keyid>]: 一番外側のIPv6ヘッダの直後に指定したSRヘッダを挿入する。オリジナルのSRヘッダは変更されない。このパケットの宛先アドレスは新しく挿入されたSRヘッダの最初のSIDで、そこに向けて転送

- End.B6.Encaps srh segs <segments> [hmac <keyid>]: パケットを次のSIDに進める（segments left値を減らし、宛先を更新する）。次に指定されたSRヘッダに従ってアウターIPv6ヘッダにカプセル化する。アウターIPv6ヘッダの宛先アドレスは指定されたSRヘッダの最初のSIDに設定される。
