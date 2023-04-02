# SRv6 Linux設定

> 参考
>
> https://segment-routing.org/


実機では試してません。

メモです。

### インタフェース単位でのSRv6の有効化設定

```
net.ipv6.conf.*.seg6_enabled (integer)
```

- 0 はSRパケットを破棄
- 1 はSRパケットを受信して処理


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
```

table 100をlocalsidとして作成。

/etc/iproute2/rt_tablesには以下の通り、既定で設定されている。

通常のルーティングテーブルは254 mainを利用する。

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

```
# ip -6 rule add to fc00::/64 lookup localsid
```

ルールを追加して宛先 fc00::/64 については localsid テーブルを参照するように設定。

```
# ip -6 route add blackhole default table localsid
```

SID以外の知らない宛先は破棄するエントリをlocalsidテーブルに加える。



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



# srext

SRv6の先進的な機能はsrextに実装されている。

> 参考
>
> https://github.com/netgroup/SRv6-net-prog


コンパイルするための環境を整える。

```
apt install make
apt install gcc
```

srextをgitで取得する。

```
git clone https://github.com/netgroup/SRv6-net-prog
```

srextディレクトリに移動する。

```
cd srv6-net-prog/srext/
```

x509.genkeyファイルを新規に作成する。

```
vi x509.genkey
```

内容はこう。

```
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts

[ req_distinguished_name ]
CN = Modules

[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
```

キーを作成する。

```
openssl req -new -nodes -utf8 -sha512 -days 36500 -batch -x509 -config x509.genkey -outform DER -out signing_key.x509 -keyout signing_key.pem
```

出来上がったキーをコピーする。

```
cp signing_key.* /lib/modules/$(uname -r)/build/certs/
```

コンパイルする。

```
make
```

インストールする。

```
make install
```

カーネルにロードする。

```
sudo depmod -a
sudo modprobe srext
```

画面には何も表示されない。

もう一度、--first-timeを付けて実行して、エラーがでればロードに成功している。

```
modprobe --first-time srext
modprobe: ERROR: could not insert 'srext': Module already in kernel
```

使い方

```
root@pe3:~/SRv6-net-prog/srext# srconf localsid
Usage: srconf localsid { help | flush }
       srconf localsid { show | clear-counters } [SID]
       srconf localsid del SID
       srconf localsid add SID BEHAVIOUR
BEHAVIOUR:= { end |
              end.dx2 TARGETIF |
              end.dx4 NEXTHOP4 TARGETIF |
              { end.x | end.dx6 } NEXTHOP6 TARGETIF |
              { end.ad4 | end.ead4 } NEXTHOP4 TARGETIF SOURCEIF |
              { end.am | end.ad6 | end.ead6 } NEXTHOP6 TARGETIF SOURCEIF |
              end.as4 NEXTHOP4 TARGETIF SOURCEIF src ADDR segs SIDLIST left SEGMENTLEFT }
              end.as6 NEXTHOP6 TARGETIF SOURCEIF src ADDR segs SIDLIST left SEGMENTLEFT |
NEXTHOP4:= { ip IPv4-ADDR | mac MAC-ADDR }
NEXTHOP6:= { ip IPv6-ADDR | mac MAC-ADDR }
```

localsidテーブルにSIDを追加する

```
srconf localsid add SID end.am ip IPv6-ADDR TARGETIF SOURCEIF
```

- SID 追加したいSID
- end.adはダイナミックプロキシ
- end.amはマスカレード
- IPv6-ADDRはプロキシ先となるVNFサービスのIPv6アドレス
- TARGETIFはプロキシ先となるVNFサービスに向けたインタフェース
- SOURCEIFはプロキシ先から戻ってくる通信を受け取るインタフェース


テーブルの確認する

```
srconf localsid show
```


入り口ノード

ingress.sh

```
#!/bin/bash

# The vagrant provisioning script for ingress node

# Install required softwares
export DEBIAN_FRONTEND=noninteractive
apt-get -y --force-yes install ethtool
apt-get -y --force-yes install iperf
apt-get -y --force-yes install iperf3
apt-get -y --force-yes install libpcap-dev

# Install latest tcpdump that support SR
git clone https://github.com/the-tcpdump-group/tcpdump
cd tcpdump/
./configure
sudo make && sudo make install && cd ..

# Enable IPv6 forwarding
sysctl -w net.ipv6.conf.all.forwarding=1

# Configure interfaces
ifconfig eth1 up
ip -6 addr add 1:2::1/64 dev eth1

# Configure routing
ip -6 route add 1:2::/64 via 1:2::2

# Instal srext
git clone https://github.com/netgroup/SRv6-net-prog
cd SRv6-net-prog/srext/
sudo make && sudo make install && depmod -a

exit
```

出口ノード

egress.sh

```
#!/bin/bash

# The vagrant provisioning script for ingress node

# Install required softwares
export DEBIAN_FRONTEND=noninteractive
apt-get -y --force-yes install ethtool
apt-get -y --force-yes install iperf
apt-get -y --force-yes install iperf3
apt-get -y --force-yes install libpcap-dev

# Install latest tcpdump that support SR
git clone https://github.com/the-tcpdump-group/tcpdump
cd tcpdump/
sudo ./configure
sudo make && sudo make install && cd ..

# Enable IPv6 forwarding
sysctl -w net.ipv6.conf.all.forwarding=1

# Configure interfaces
ifconfig eth1 up
ip -6 addr add 2:3::3/64 dev eth1

# Configure routing
ip -6 route add 2::/64 via 2:3::2

# Instal srext
git clone https://github.com/netgroup/SRv6-net-prog
cd SRv6-net-prog/srext/
sudo make && sudo make install && depmod -a

exit
```


nfvノード

```
#!/bin/bash

# The vagrant provisioning script for ingress node

# Install required softwares
export DEBIAN_FRONTEND=noninteractive
apt-get -y --force-yes install ethtool
apt-get -y --force-yes install iperf
apt-get -y --force-yes install iperf3
apt-get -y --force-yes install libpcap-dev

# Install latest tcpdump that support SR
git clone https://github.com/the-tcpdump-group/tcpdump
cd tcpdump/
sudo ./configure
sudo make && sudo make install && cd ..

# Enable IPv6 forwarding
sysctl -w net.ipv6.conf.all.forwarding=1

# Configure interfaces
ifconfig eth1 up
ip -6 addr add 1:2::2/64 dev eth1

ifconfig eth2 up
ip -6 addr add 2:3::2/64 dev eth2

# Configure routing
ip -6 route add 1::/64 via 1:2::1
ip -6 route add 3::/64 via 2:3::3

# Instal srext
git clone https://github.com/netgroup/SRv6-net-prog
cd SRv6-net-prog/srext/
sudo make && sudo make install && depmod -a

exit

```