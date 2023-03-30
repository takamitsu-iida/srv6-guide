# Ubuntu Server 22.04 L3VPN over SRv6

LinuxだけでL3VPNを構成します。


## 構成

![構成](img/frr_l3vpn.drawio.png)


## EVE-NG

仮想基盤としてEVE-NGを利用します。

Ubuntuのイメージは出来合いのものを利用します。

> 参考
>
> https://www.eve-ng.net/index.php/documentation/howtos/howto-create-own-linux-host-image/

Ubuntu自体アップデートやソフトウェアのインストールにインターネット接続が必要になります。
インターネットには母艦となるホストマシンのNATを利用して接続するようにします。

![EVE-NG](img/frr-eve-ng.png)


### Ubuntuイメージ起動時のQEMU設定

QEMUのオプションに `-k ja` を追加してから起動するようにします。

日本語キーボードを使う、という指定です。

![構成](img/frr-qemu.png)

この設定は結線したりラボに変更を加えると消えてしまうことがあるので、その都度、追加します。


### Ubuntuのrootパスワード変更

出来合いのイメージのrootのパスワードはTest123になっていますので、これを変更します。

VNCクライアントで接続し、rootでログインします。

```bash
passwd root
```

### キーボードの設定を変更

そのままだとviで編集作業をするときにコロンが打てなくて難儀しますので、日本語キーボードに変更します。

```bash
dpkg-reconfigure keyboard-configuration
```

表示される選択肢に従って`Japanese`を選びます。


### シリアルコンソールの有効化

Ubuntu ServerなのでGUIは不要なのですが、そのままだとVNCクライアントが起動してしまいます。

シリアルコンソールを動かしてtelnet接続するように変更します。

`vi /etc/default/grub`

GRUB_CMD_LINE_LINUX=""となっている部分を以下のように変更します。

```
GRUB_CMDLINE_LINUX="console=ttyS0,115200 console=tty0"
```

編集したら反映させます。

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Ubuntuを停止します。

```bash
shutdown –h now
```

EVE-NGの設定でコンソール接続をtelnetに変更します。

Ubuntuを起動します。

### Ubuntuのホスト名を変更

4台のUbuntuにそれぞれhostnameを設定します。

```bash
hostnamectl set-hostname <ホスト名>
```

### Ubuntuのタイムゾーンを変更

```bash
timedatectl set-timezone Asia/Tokyo
```

### Ubuntuのプロキシを設定（必要なら）

認証プロキシを経由しないとインターネット接続できない環境の場合は、各Ubuntuにプロキシの設定を行います。

`vi ~/.bashrc`

```
export http_proxy="http://username:password@proxy-address:8080"
export https_proxy="http://username:password@proxy-address:8080"
```

`source ~/.bashrc`

aptコマンドが参照するプロキシも設定しておきます。

`vi /etc/apt/apt.conf`

```
Acquire::ftp::proxy "http://username:password@proxy-address:8080";
Acquire::http::proxy  "http://username:password@proxy-address:8080";
Acquire::https::proxy "http://username:password@proxy-address:8080";
```

### Ubuntuのアップデート

Ubuntu自体をアップデートします。

```bash
apt update
```

シリアルコンソールでviを使うとその後、画面が乱れますのでresizeコマンドを使うためにxtermをインストールします。

```bash
apt install xterm
```

`resize -c` コマンドで画面がキレイになります。


### Ubuntuのイーサネットインタフェース設定

EVE-NGの設定で追加したNICはダウン状態なので、これを手動でアップさせます。

```bash
ip link set ens4 up
ip link set ens5 up
ip link set ens6 up
```

起動時にDHCPでアドレスを取得すると時間がかかるので、DHCPを使わないように設定を変更します。

ついでにインタフェースの名前もEVE-NGで認識しているものと一致するように変更します。

> 参考
>
> https://ubuntu.com/server/docs/network-configuration

ip aでMAC Addressを調べて、そのMACアドレスにマッチしたらset-nameで名前を変更します。

`cd /etc/netplan`

`vi 00-installer-config.yaml`

```yaml
network:
  ethernets:
    e0:
      dhcp4: true
      match:
        macaddress: 50:02:00:09:00:00
      set-name: e0
    e1:
      dhcp4: false
      match:
        macaddress: 50:02:00:09:00:01
      set-name: e1
    e2:
      dhcp4: false
      match:
        macaddress: 50:02:00:09:00:02
      set-name: e2
    e3:
      dhcp4: false
      match:
        macaddress: 50:02:00:09:00:03
      set-name: e3
  version: 2
```

macaddressの部分はUbuntuごとに変わります。

e0は母艦となるホストマシンのNATインタフェースですので、DHCPでアドレスを取得します。
それ以外は全てDHCPを無効にします。


### カーネルのパラメータ変更

Ubuntuをルータとして動けるようにするほか、VRFとSRv6のパラメータも有効にします。

`vi /etc/sysctl.d/99-sysctl.conf`

```ini
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
net.ipv6.conf.all.forward=1

# Reverse Path Filter
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0

# IPv6 address
net.ipv6.conf.all.keep_addr_on_down=1

# SRv6
net.ipv6.conf.all.seg6_enabled=1
net.ipv6.seg6_flowlabel=1

# VRF
net.ipv4.tcp_l3mdev_accept=1
net.ipv4.udp_l3mdev_accept=1
net.ipv4.raw_l3mdev_accept=1
```


## FRRのインストール

動的ルーティングのデーモンとしてFRRを利用します。

> 参考
>
> https://frrouting.org/
>
> https://deb.frrouting.org/

Downloadをクリックしてインストール方法を確認します。

以下の通りに実行します。

ここではfrr-stableを選んでいます。

> 参考
>
> 2023年3月時点でstableはバージョン8.5です

```bash
# add GPG key
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -

# possible values for FRRVER: frr-6 frr-7 frr-8 frr-stable
# frr-stable will be the latest official stable release
FRRVER="frr-stable"
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER | sudo tee -a /etc/apt/sources.list.d/frr.list

# update and install FRR
sudo apt update && sudo apt install frr frr-pythontools
```

BGPとISISを有効にします。

`vi /etc/frr/daemons`

```
# When using "vtysh" such a config file is also needed. It should be owned by
# group "frrvty" and set to ug=rw,o= though. Check /etc/pam.d/frr, too.
#
# The watchfrr, zebra and staticd daemons are always started.
#
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=yes
pimd=no
pim6d=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=yes
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
```

変更点はこの２つです。
- bgpd=yes
- isisd=yes

`reboot` で再起動します。


## VRFの作成

FRRではVRFを操作できませんので、Ubuntu側で事前に作成します。

> 参考
>
> https://docs.kernel.org/networking/vrf.html

/nic/binにVRF作成用のシェルスクリプトを用意します。

```bash
mkdir -p /nic
mkdir -p /nic/bin
chmod 755 /nic/bin
```

vi /nic/bin/create_vrf.sh

作成するVRFはPEルータによって異なりますが、概ね次のような内容になります。

```bash
#!/bin/sh

# VRF vrfA を作成する
# テーブル番号は一意であれば何でもよい
/usr/sbin/ip link add vrfA type vrf table 110

# e3にVLANのサブインタフェースを追加する
# /usr/sbin/ip link add link e3 name e3.110 type vlan id 110

# e3をvrfAに組み込む
/usr/sbin/ip link set dev e3 master vrfA up

# vrfAを有効にする
/usr/sbin/ip link set dev vrfA up

# VRFのstrict_modeは必ず1にします
/usr/sbin/sysctl -w net.vrf.strict_mode=1

# 作成したVRFのReverse Path Filterを停止
/usr/sbin/sysctl -w net.ipv4.conf.vrfA.rp_filter=0

exit 0
```

```bash
chown root:root /nic/bin/create_vrf.sh
chmod 755 /nic/bin/create_vrf.sh
```

作成したVRFは永続ではなく、再起動すると消えてしまいます。

起動時に自動でVRFを作るためにはスクリプトを作成してsystemdから呼び出すようにします。

/etc/systemd/systemディレクトリに.serviceで終わる設定ファイルを用意します。

名前は任意です。ここではcreate_vrf.serviceとします。

`vi /etc/systemd/system/create_vrf.service`

```ini
[Unit]
Description = create vrf
After=local-fs.target networking.service
ConditionPathExists=/nic/bin

[Service]
ExecStart=/nic/bin/create_vrf.sh
Restart=no
Type=simple

[Install]
WantedBy=multi-user.target
```

呼び出すスクリプトは前述の`/nic/bin/create_vrf.sh`です。

```bash
chown root:root /etc/systemd/system/create_vrf.service
chmod 644 /etc/systemd/system/create_vrf.service
```

スクリプトを配置しただけでは自動起動しませんので`systemctl`コマンドで登録します。

```bash
systemctl daemon-reload
systemctl enable create_vrf.service
```

- 【参考】起動と停止方法

```bash
systemctl start ユニットファイル名
systemctl stop ユニットファイル名
```

### 【参考】ipコマンド

- 【参考】VRFの情報を表示する方法

```bash
ip [-d] link show type vrf
ip [-d] link show vrf NAME
ip [-d] link show master NAME
```

実行例

```bash
root@pe3:~# ip -d link show type vrf
6: vrfA: <NOARP,MASTER,UP,LOWER_UP> mtu 65575 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 8e:37:5c:e7:86:fd brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 1280 maxmtu 65575
    vrf table 110 addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
```

- 【参考】VRFのアドレス情報を表示する方法

```bash
ip addr show vrf NAME
ip addr show master NAME
```

- 【参考】VRFのルーティングテーブルを表示する方法

```bash
ip [-6] route show vrf NAME
ip [-6] route show table ID
```

- 【参考】インタフェースとVRFの紐づけを解除する方法

```bash
ip link set dev NAME nomaster
```

- 【参考】スタティックルートを設定する方法

```bash
ip route add 0.0.0.0/0 via 192.168.3.5 vrf vrfA
```

- 【参考】pingする方法

-Iにvrf名を指定します。

```bash
ping -I vrfA 192.168.3.5
```

もしくは `ip vrf exec vrf名` を付けて実行します。

```bash
ip vrf exec vrfA ping 192.168.3.5
```


## FRRの設定

rootでログイン後、vtyshを起動して、以下のように設定します。

- PE3の設定

```
pe3# sh run
Building configuration...

Current configuration:
!
frr version 8.5
frr defaults traditional
hostname pe3
log syslog informational
service integrated-vtysh-config
!
interface lo
 ipv6 address 2001:db8:0:3::1/128
 ipv6 router isis core
exit
!
interface e3
 ip address 192.168.3.1/24
exit
!
interface e1
 ipv6 router isis core
 isis metric 1
 isis network point-to-point
exit
!
interface e2
 ipv6 router isis core
 isis metric 2
 isis network point-to-point
exit
!
router bgp 65001
 bgp router-id 0.0.0.3
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 neighbor RR peer-group
 neighbor RR remote-as 65001
 neighbor RR update-source 2001:db8:0:3::1
 neighbor RR capability extended-nexthop
 neighbor RR graceful-restart
 neighbor 2001:db8:0:1::1 peer-group RR
 neighbor 2001:db8:0:2::1 peer-group RR
 !
 segment-routing srv6
  locator a
 exit
 !
 address-family ipv4 vpn
  neighbor RR activate
  neighbor RR soft-reconfiguration inbound
 exit-address-family
 !
 address-family ipv6 unicast
  network 2001:db8:0:3::1/128
  aggregate-address 2001:db8:0:3::/64 summary-only
  neighbor RR activate
  neighbor RR soft-reconfiguration inbound
 exit-address-family
exit
!
router bgp 65001 vrf vrfA
 !
 address-family ipv4 unicast
  redistribute connected
  sid vpn export auto
  rd vpn export 65001:110
  nexthop vpn export 2001:db8:0:3::1
  rt vpn both 65001:110
  export vpn
  import vpn
 exit-address-family
exit
!
router isis core
 net 49.0000.0000.0000.0003.00
 topology ipv6-unicast
exit
!
segment-routing
 srv6
  locators
   locator a
    prefix 2001:db8:0:3::/64 block-len 40 node-len 24 func-bits 16
   exit
   !
  exit
  !
 exit
 !
exit
!
end
```

- PE4の設定

```
pe4# show run
Building configuration...

Current configuration:
!
frr version 8.5
frr defaults traditional
hostname pe4
log syslog informational
service integrated-vtysh-config
!
interface lo
 ipv6 address 2001:db8:0:4::1/128
 ipv6 router isis core
exit
!
interface e3
 ip address 192.168.4.1/24
exit
!
interface e1
 ipv6 router isis core
 isis metric 1
 isis network point-to-point
exit
!
interface e2
 ipv6 router isis core
 isis metric 2
 isis network point-to-point
exit
!
router bgp 65001
 bgp router-id 0.0.0.4
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 neighbor RR peer-group
 neighbor RR remote-as 65001
 neighbor RR update-source 2001:db8:0:4::1
 neighbor RR capability extended-nexthop
 neighbor RR graceful-restart
 neighbor 2001:db8:0:1::1 peer-group RR
 neighbor 2001:db8:0:2::1 peer-group RR
 !
 segment-routing srv6
  locator a
 exit
 !
 address-family ipv4 vpn
  neighbor RR activate
  neighbor RR soft-reconfiguration inbound
 exit-address-family
 !
 address-family ipv6 unicast
  network 2001:db8:0:4::1/128
  aggregate-address 2001:db8:0:4::/64 summary-only
  neighbor RR activate
  neighbor RR soft-reconfiguration inbound
 exit-address-family
exit
!
router bgp 65001 vrf vrfA
 !
 address-family ipv4 unicast
  redistribute connected
  sid vpn export auto
  rd vpn export 65001:110
  nexthop vpn export 2001:db8:0:4::1
  rt vpn both 65001:110
  export vpn
  import vpn
 exit-address-family
exit
!
router isis core
 net 49.0000.0000.0000.0004.00
 topology ipv6-unicast
exit
!
segment-routing
 srv6
  locators
   locator a
    prefix 2001:db8:0:4::/64 block-len 40 node-len 24 func-bits 16
   exit
   !
  exit
  !
 exit
 !
exit
!
end
```

- CR1の設定

```
cr1# show run
Building configuration...

Current configuration:
!
frr version 8.5
frr defaults traditional
hostname cr1
log syslog informational
service integrated-vtysh-config
!
interface lo
 ipv6 address 2001:db8:0:1::1/128
 ipv6 router isis core
exit
!
interface e1
 ipv6 router isis core
 isis metric 1
 isis network point-to-point
exit
!
interface e2
 ipv6 router isis core
 isis metric 1
 isis network point-to-point
exit
!
interface e3
 ipv6 router isis core
 isis metric 100
 isis network point-to-point
exit
!
router bgp 65001
 bgp router-id 0.0.0.1
 bgp log-neighbor-changes
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 bgp cluster-id 0.0.0.1
 bgp graceful-restart
 neighbor CLIENT peer-group
 neighbor CLIENT remote-as 65001
 neighbor CLIENT update-source lo
 neighbor CLIENT capability extended-nexthop
 neighbor CLIENT graceful-restart
 neighbor CLUSTER peer-group
 neighbor CLUSTER remote-as 65001
 neighbor CLUSTER description route_reflector_cluster_peer
 neighbor CLUSTER update-source lo
 neighbor CLUSTER capability extended-nexthop
 neighbor CLUSTER graceful-restart
 neighbor 2001:db8:0:3::1 peer-group CLIENT
 neighbor 2001:db8:0:4::1 peer-group CLIENT
 neighbor 2001:db8:0:2::1 peer-group CLUSTER
 !
 address-family ipv4 vpn
  neighbor CLIENT activate
  neighbor CLIENT route-reflector-client
  neighbor CLIENT soft-reconfiguration inbound
  neighbor CLUSTER activate
  neighbor CLUSTER soft-reconfiguration inbound
 exit-address-family
 !
 address-family ipv6 unicast
  network 2001:db8:0:1::1/128
  aggregate-address 2001:db8:0:1::/64 summary-only
  neighbor CLIENT activate
  neighbor CLIENT route-reflector-client
  neighbor CLIENT soft-reconfiguration inbound
  neighbor CLUSTER activate
  neighbor CLUSTER soft-reconfiguration inbound
 exit-address-family
exit
!
router isis core
 net 49.0000.0000.0000.0001.00
 topology ipv6-unicast
exit
!
segment-routing
 srv6
  locators
   locator a
    prefix 2001:db8:0:1::/64 block-len 40 node-len 24 func-bits 16
   exit
   !
  exit
  !
 exit
 !
exit
!
end
```

- CR2の設定

```
cr2# sh run
Building configuration...

Current configuration:
!
frr version 8.5
frr defaults traditional
hostname cr2
log syslog informational
service integrated-vtysh-config
!
interface lo
 ipv6 address 2001:db8:0:2::1/128
 ipv6 router isis core
exit
!
interface e1
 ipv6 router isis core
 isis metric 100
 isis network point-to-point
exit
!
interface e2
 ipv6 router isis core
 isis metric 1
 isis network point-to-point
exit
!
interface e3
 ipv6 router isis core
 isis metric 1
 isis network point-to-point
exit
!
router bgp 65001
 bgp router-id 0.0.0.2
 bgp log-neighbor-changes
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 bgp cluster-id 0.0.0.1
 bgp graceful-restart
 neighbor CLIENT peer-group
 neighbor CLIENT remote-as 65001
 neighbor CLIENT update-source lo
 neighbor CLIENT capability extended-nexthop
 neighbor CLIENT graceful-restart
 neighbor CLUSTER peer-group
 neighbor CLUSTER remote-as 65001
 neighbor CLUSTER update-source lo
 neighbor CLUSTER capability extended-nexthop
 neighbor CLUSTER graceful-restart
 neighbor 2001:db8:0:3::1 peer-group CLIENT
 neighbor 2001:db8:0:4::1 peer-group CLIENT
 neighbor 2001:db8:0:1::1 peer-group CLUSTER
 !
 address-family ipv4 vpn
  neighbor CLIENT activate
  neighbor CLIENT route-reflector-client
  neighbor CLIENT soft-reconfiguration inbound
  neighbor CLUSTER activate
  neighbor CLUSTER soft-reconfiguration inbound
 exit-address-family
 !
 address-family ipv6 unicast
  network 2001:db8:0:2::1/128
  aggregate-address 2001:db8:0:2::/64 summary-only
  neighbor CLIENT activate
  neighbor CLIENT route-reflector-client
  neighbor CLIENT soft-reconfiguration inbound
  neighbor CLUSTER activate
  neighbor CLUSTER soft-reconfiguration inbound
 exit-address-family
exit
!
router isis core
 net 49.0000.0000.0000.0002.00
 topology ipv6-unicast
exit
!
segment-routing
 srv6
  locators
   locator a
    prefix 2001:db8:0:2::/64 block-len 40 node-len 24 func-bits 16
   exit
   !
  exit
  !
 exit
 !
exit
!
end
```


## 設定の解説

網内のインタフェースに割り当てるIPv6アドレスはリンクローカルのみです。本来なら付与した方がよいでしょう。

各装置に/64のロケータを割り当てています。

- CR1のロケータは `2001:db8:0:1::/64`
- CR2のロケータは `2001:db8:0:2::/64`
- PE3のロケータは `2001:db8:0:3::/64`
- PE3のロケータは `2001:db8:0:4::/64`

ロケータの設定はこの部分です（PE3の場合）。

```
!
segment-routing
 srv6
  locators
   locator a
    prefix 2001:db8:0:3::/64 block-len 40 node-len 24 func-bits 16
   exit
   !
  exit
  !
 exit
 !
exit
!
```

`a` はロケータの名前です。装置の中でユニークなら何でも構いません。
block-len 40などは勝手に付与されたもので、実際に入力したのは`prefix 2001:db8:0:2::/64`だけです。

ループバックインタフェース `lo` にはロケータの中からIPv6アドレスを採番します。
それがこの部分（PE3の場合）。

```
!
interface lo
 ipv6 address 2001:db8:0:3::1/128
 ipv6 router isis core
exit
!
```

PE3のロケータが `2001:db8:0:3::/64` なので、その中から末尾が1の/128のアドレスを採番しています。

困ったことに **FRRのISISはロケータのプレフィクス情報を配信しません** ので、
何らかの方法で全ての装置にプレフィクス情報を教えてあげる必要があります。

ここではBGPを使ってロケータの情報を配信します。それがこの部分（PE3の場合）。

```
!
router bgp 65001
 !
 address-family ipv6 unicast
  network 2001:db8:0:3::1/128
  aggregate-address 2001:db8:0:3::/64 summary-only
 exit-address-family
exit
!
```

ループバックインタフェース `lo` に付けた/128アドレスを/64に集約して配信することで、ロケータのプレフィクスを配信します。

PE3においてIPv6のルーティングテーブルを表示するとこのようになります。

```
pe3# show ipv6 route
Codes: K - kernel route, C - connected, S - static, R - RIPng,
       O - OSPFv3, I - IS-IS, B - BGP, N - NHRP, T - Table,
       v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>d ::/0 [0/1024] via fe80::5202:ff:fe09:1, e1, weight 1, 01:25:34
  d               via fe80::5202:ff:fe0a:2, e2, weight 1, 01:25:34
B>  2001:db8:0:1::/64 [200/0] via 2001:db8:0:1::1 (recursive), weight 1, 01:25:50
  *                             via fe80::5202:ff:fe09:1, e1, weight 1, 01:25:50
I>* 2001:db8:0:1::1/128 [115/11] via fe80::5202:ff:fe09:1, e1, weight 1, 01:25:51
B>  2001:db8:0:2::/64 [200/0] via 2001:db8:0:2::1 (recursive), weight 1, 00:04:22
  *                             via fe80::5202:ff:fe0a:2, e2, weight 1, 00:04:22
I>* 2001:db8:0:2::1/128 [115/12] via fe80::5202:ff:fe0a:2, e2, weight 1, 00:04:22
B>* 2001:db8:0:3::/64 [200/0] unreachable (blackhole), weight 1, 16:23:03
C>* 2001:db8:0:3::1/128 is directly connected, lo, 16:23:03
B>* 2001:db8:0:3:1::/128 [20/0] is directly connected, vrfA, seg6local End.DT4 table 110, seg6 ::, weight 1, 16:23:03
B>  2001:db8:0:4::/64 [200/0] via 2001:db8:0:4::1 (recursive), weight 1, 01:25:50
  *                             via fe80::5202:ff:fe09:1, e1, weight 1, 01:25:50
I>* 2001:db8:0:4::1/128 [115/12] via fe80::5202:ff:fe09:1, e1, weight 1, 01:25:51
C * fe80::/64 is directly connected, e2, 16:23:04
C * fe80::/64 is directly connected, e1, 16:23:04
C>* fe80::/64 is directly connected, e0, 16:23:04
```

たくさん表示されて見づらいので、/64の経路だけを表示してみます。

```
root@pe3:/nic/bin# vtysh -c 'show ipv6 route' | grep '/64'
B>  2001:db8:0:1::/64 [200/0] via 2001:db8:0:1::1 (recursive), weight 1, 01:34:03
B>  2001:db8:0:2::/64 [200/0] via 2001:db8:0:2::1 (recursive), weight 1, 00:12:35
B>* 2001:db8:0:3::/64 [200/0] unreachable (blackhole), weight 1, 16:31:16
B>  2001:db8:0:4::/64 [200/0] via 2001:db8:0:4::1 (recursive), weight 1, 01:34:03
C * fe80::/64 is directly connected, e2, 16:31:17
C * fe80::/64 is directly connected, e1, 16:31:17
C>* fe80::/64 is directly connected, e0, 16:31:17
```

各装置に割り当てたロケータのプレフィクス（/64）がBGPで配信されて学習できています。


## VRFのルーティングテーブル

PE3において、vrfAに関するルーティングテーブルを表示してみます。

```
pe3# show ip route vrf vrfA
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

VRF vrfA:
C>* 192.168.3.0/24 is directly connected, e3, 17:34:41
B>  192.168.4.0/24 [200/0] via 2001:db8:0:4::1 (vrf default) (recursive), label 16, seg6local unspec unknown(seg6local_context2str), seg6 2001:db8:0:4:1::, weight 1, 02:37:28
  *                          via fe80::5202:ff:fe09:1, e1 (vrf default), label 16, seg6local unspec unknown(seg6local_context2str), seg6 2001:db8:0:4:1::, weight 1, 02:37:28
pe3#
```

192.168.3.0/24は自足のインタフェースe3に割り当てたものです。

192.168.4.0/24は対向するPE4側のアドレスです。

viaとなっている2001:db8:0:4::1はPE4のループバックアドレスです。

label 16は意味不明。

seg6 2001:db8:0:4:1::はSRv6のSIDです。


## PE3から送信されるパケット


![ICMP](img/ping_from_ce5_to_ce6.png)

送信元アドレス（2001:db8:0:3::1）はPE3のループバックアドレスです。

送信先アドレス（2001:db8:0:4:1::）はPE4のEnd.DT4のSIDです。

> pcapngファイル
>
> img/ping_from_ce5_to_ce6.pcapng
>
>