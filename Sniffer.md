# スニッファーについて

「スニッファー」とはトラフィックを監視してパケットの解析などを行うソフトウェアのこと。

```python
import socket
import os

HOST = '192.168.2.103'

def main():    
    if os.name == 'nt':
        socket_protocol = socket.IPPROTO_IP
    else:       
        socket_protocol = socket.IPPROTO_ICMP

    sniffer = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket_protocol)
    sniffer.bind((HOST, 0))
    
    sniffer.setsockopt(socket.IPPROTO_IP, socket.IP_HDRINCL, 1)

    if os.name == 'nt':        
        sniffer.ioctl(socket.SIO_RCVALL, socket.RCVALL_ON)
    
    print(sniffer.recvfrom(65565))
        
    if os.name == 'nt':
        sniffer.ioctl(socket.SIO_RCVALL, socket.RCVALL_OFF)

if __name__ == '__main__':
    main()
```

リッスンするホストとして自分のIPアドレスを変数`HOST`に設定する。

ソケットオブジェクトに与える引数については、正直「こういうものなんだな〜」くらいで書き写してしまっていいと個人的には思う。

```python
sniffer = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket_protocol)
sniffer.bind((HOST, 0))
sniffer.setsockopt(socket.IPPROTO_IP, socket.IP_HDRINCL, 1)
```

スニッフする時にソケットオブジェクトに与える引数はコレ、IPヘッダー情報欲しいときはコレを引数として与えるって感じで。

スニッファーとして”プロミスキャスモード”で動作する。プロミスキャスモードとは、自分宛でないパケットでも全部受け取るモードのこと。

```python
if os.name == 'nt':
    sniffer.ioctl(socket.SIO_RCVALL, socket.RCVALL_ON)
```

Widowsでプロミスキャスモードを使用するにはIOCTL値なるものの設定が必要で、それがこの部分。

# IPのデコード

やってみるとわかるが、上のスクリプトではレスポンスがバイナリのままで読みにくい。

そこでPythonの`struct`モジュールを使ってデコードの機能を実装する。

IPヘッダの情報を扱うクラス`IP`を作成

```python
import ipaddress
import struct

class IP:
    def __init__(self, buff = None):        
        header = struct.unpack('<BBHHHBBH4s4s', buff)
        self.ver = header[0] >> 4
        self.ihl = header[0] & 0xF

        self.tos = header[1]
        self.len = header[2]
        self.id = header[3]
        self.offset = header[4]
        self.ttl = header[5]
        self.protocol_num = header[6]
        self.sum = header[7]
        self.src = header[8]
        self.dst = header[9]

        self.src_address = ipaddress.ip_address(self.src)
        self.dst_addresss = ipaddress.ip_address(self.dst)
        
        self.protocol_map = {1: "ICMP", 6: "TCP", 17: "UDP"}
```

変数`buff`はパケット等データのこと。

まずデータ`buff`を書式指定文字に従ってバイナリからか読みやすいように変換（アンパック）する。

```python
header = struct.unpack('<BBHHHBBH4s4s', buff)
```

返り値の型はタプルである。

書式指定文字については`struct`モジュールのドキュメントを参照：[struct --- バイト列をパックされたバイナリデータとして解釈する](https://docs.python.org/ja/3/library/struct.html)

そしてそのタプルの各要素がIPヘッダのそれぞれの要素に対応するから、上に示したコードの通りインスタンス変数を設定する。

![ipheader](https://user-images.githubusercontent.com/85237728/168715225-fa8b86e0-60fd-4bec-a7d5-32bb695ae136.png)

先に出てきた書式指定文字との対応はこんな感じ👇

![ipheader_with_pen](https://user-images.githubusercontent.com/85237728/168715359-89ae11dd-6f62-428c-b6bb-251ea12dfc48.png)

ちょうど書式指定文字のBやHと同じ個数のインスタンス変数があることにも注目。

```python
self.ver = header[0] >> 4
self.ihl = header[0] & 0xF
```

IPのバージョンとIPヘッダ長をそれぞれインスタンス変数`ver`と`ihl`に入れる。

```
self.ver = header[0] >> 4
```

これは`header[0]`の値を右に4ビットシフトすることを意味する。

以下に具体例を示す

```python
>>> a = 0b10101001
>>> bin(a)
'0b10101001'
>>> bin(a >> 4)
'0b1010'
```

右に4ビットシフトするということは、`10101001`の左に`0000`と`0`が4つ付く。

```python
self.src_address = ipaddress.ip_address(self.src) 
elf.dst_addresss = ipaddress.ip_address(self.dst)
```

`ipaddress`の`ip_address`は、IPアドレスオブジェクトを受け取ってそれをIPアドレスとして読みやすいにようにしたものを返す。

# ICMPのデコード

ICMPのデコードもIPのそれとほとんど同じコードで実装できる。

特に重要なこととしては、UDPパケットを打ってUnreachableが返ってくるということはそのホストがaliveしている証拠という点。

もしどのホストもaliveしていなければ何も返ってこない。

ICMPの部分については本書を参照。

# ipaddressモジュールについて

忘れないように`ipaddress`モジュールについてここで簡単にまとめておく。

```python
ipaddress.ip_address('192.168.2.105')
```

`ipaddress.ip_address`にはstr型のアドレスを渡せばアドレスオブジェクトが返ってくる。

```python 
ipaddress.ip_network('192.168.2.0/24')
```

`ipaddress.ip_network`には上のようにサブネットを渡すと、ネットワークオブジェクトを返す。

`ipaddress`モジュールの公式ドキュメント：[ipaddress --- IPv4/IPv6 操作ライブラリ](https://docs.python.org/ja/3/library/ipaddress.html)
