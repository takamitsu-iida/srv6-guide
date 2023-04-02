# SRv6 Linux srext設定

SRv6の先進的な機能はsrextに実装されています。

> 参考
>
> https://github.com/netgroup/SRv6-net-prog


# srextのインストール

コンパイル環境を準備します。

```
apt install make
apt install gcc
```

srextをgitで取得します。

```
git clone https://github.com/netgroup/SRv6-net-prog
```

srextディレクトリに移動します。

```
cd srv6-net-prog/srext/
```

x509.genkeyファイルを新規に作成します。

```
vi x509.genkey
```

内容はこのようにします。

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

opensslコマンドでキーを作成します。

```
openssl req -new -nodes -utf8 -sha512 -days 36500 -batch -x509 -config x509.genkey -outform DER -out signing_key.x509 -keyout signing_key.pem
```

出来上がったキーをコピーします。

```
cp signing_key.* /lib/modules/$(uname -r)/build/certs/
```

srextをコンパイルします。

```
make
```

srextをインストールします。

```
make install
```

srextをカーネルにロードします。

```
sudo depmod -a
sudo modprobe srext
```

画面には何も表示されないのが期待値です。

もう一度、--first-timeを付けて実行してエラーがでればロードに成功しています。

```
modprobe --first-time srext
modprobe: ERROR: could not insert 'srext': Module already in kernel
```
