# Ubuntu Server 22.04 L3VPN over SRv6

LinuxだけでL3VPNを構成します。

[SRv6 BGP](README.bgp.md) IPv4 over SRv6構成の設定例です。BGPを使ってVPNの経路情報を交換します。

[SRv6 STATIC](README.static.md) L3VPN over SRv6構成の設定例です。BGPを使わずにスタティックにSIDを設定します。

[srext](README.srext.md) カーネルモジュールsrextを使ってproxy動作を試しましたが、動作しなかった例です。参考までに記録を残します。

<br><br>

## 留意事項

- Ubuntu Server 22.04を使います

- 動的ルーティングのデーモンとしてFRRoutingを使います



<!--

EVE-NG上にUbuntu Serverを設定するための流し込みスクリプト

一部のコマンドはインタラクティブなので注意が必要。

# 要インタラクティブ
dpkg-reconfigure keyboard-configuration

ROOT_PASSWORD=eve
HOSTNAME=Linux1
PROXY_USERNAME=username
PROXY_PASSWORD=password
PROXY_ADDRESS=proxyaddress

echo "root:${ROOT_PASSWORD}" | chpasswd

hostnamectl set-hostname ${HOSTNAME}

timedatectl set-timezone Asia/Tokyo

cat - << EOS >> ~/.bashrc
export http_proxy="http://${PROXY_USERNAME}:${PROXY_PASSWORD}@${PROXY_ADDRESS}:8080"
export https_proxy="http://${PROXY_USERNAME}:${PROXY_PASSWORD}@${PROXY_ADDRESS}:8080"
EOS

source ~/.bashrc

cat - << EOS >> /etc/apt/apt.conf
Acquire::ftp::proxy "http://${PROXY_USERNAME}:${PROXY_PASSWORD}@${PROXY_ADDRESS}:8080";
Acquire::http::proxy  "http://${PROXY_USERNAME}:${PROXY_PASSWORD}@${PROXY_ADDRESS}:8080";
Acquire::https::proxy "http://${PROXY_USERNAME}:${PROXY_PASSWORD}@${PROXY_ADDRESS}:8080";
EOS

apt update

# 要インタラクティブ
apt -y upgrade

apt -y autoremove

apt -y install xterm

ip link set ens4 up
ip link set ens5 up
ip link set ens6 up

cat - << EOS > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    e0:
      dhcp4: true
      match:
        macaddress: __:__:__:__:__:e0
      set-name: e0

    e1:
      dhcp4: false
      match:
        macaddress: __:__:__:__:__:e1
      set-name: e1

    e2:
      dhcp4: false
      match:
        macaddress: __:__:__:__:__:e2
      set-name: e2

    e3:
      dhcp4: false
      match:
        macaddress: __:__:__:__:__:e3
      set-name: e3

  version: 2
EOS

e0mac=`ip link list ens3 | grep ether | cut -d\  -f 6`
e1mac=`ip link list ens4 | grep ether | cut -d\  -f 6`
e2mac=`ip link list ens5 | grep ether | cut -d\  -f 6`
e3mac=`ip link list ens6 | grep ether | cut -d\  -f 6`

sed -i "s/macaddress: __:__:__:__:__:e0/macaddress: ${e0mac}/" /etc/netplan/00-installer-config.yaml
sed -i "s/macaddress: __:__:__:__:__:e1/macaddress: ${e1mac}/" /etc/netplan/00-installer-config.yaml
sed -i "s/macaddress: __:__:__:__:__:e2/macaddress: ${e2mac}/" /etc/netplan/00-installer-config.yaml
sed -i "s/macaddress: __:__:__:__:__:e3/macaddress: ${e3mac}/" /etc/netplan/00-installer-config.yaml

sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="console=ttyS0,115200 console=tty0"/' /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg

cat - << EOS >> /etc/sysctl.d/99-sysctl.conf

# IPv4 packet forwarding
net.ipv4.ip_forward=1

# IPv6 packet forwarding
net.ipv6.conf.all.forward=1

# Reverse Path Filter
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0

# IPv6 address
net.ipv6.conf.all.keep_addr_on_down=1

# SRv6
net.ipv6.conf.all.seg6_enabled=1
net.ipv6.conf.all.seg6_require_hmac=0
net.ipv6.conf.default.seg6_enabled=1
net.ipv6.seg6_flowlabel=1

# VRF
net.ipv4.tcp_l3mdev_accept=1
net.ipv4.udp_l3mdev_accept=1
net.ipv4.raw_l3mdev_accept=1
EOS

# FRR install

# add GPG key
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -

# possible values for FRRVER: frr-6 frr-7 frr-8 frr-stable
# frr-stable will be the latest official stable release
FRRVER="frr-stable"
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER | sudo tee -a /etc/apt/sources.list.d/frr.list

# update and install FRR
sudo apt update && sudo apt -y install frr frr-pythontools

sed -i "s/bgpd=no/bgpd=yes/" /etc/frr/daemons
sed -i "s/isisd=no/isisd=yes/" /etc/frr/daemons

-->