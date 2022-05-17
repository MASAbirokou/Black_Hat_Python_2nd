# TCPクライアント

```python
 import socket

HOST = 'www.google.com'
PORT = 80

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect((HOST, PORT))

client.send(b'GET / HTTP/1.1\r\nHost: google.com\r\n\r\n')
response = client.recv(4096)
print(response.decode('utf-8'))

client.close()
```

socketオブジェクトを使った典型的なソケットプログラミング。

公式ドキュメント：[ソケットプログラミング HOWTO](https://docs.python.org/ja/3/howto/sockets.html)

まずは

```python
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

でソケットオブジェクトを作る。

これはお決まりの引数で

- AF_INET…IPv4の意味
- SOCK_STREAM…TCPクライアントの意味

ソケットオブジェクトを作ったら、ホストとポート番号のタプルを引数にしてTCPクライアントをサーバーに接続する。

```python
client.connect((HOST, PORT))
```

受け取ったデータを表示する。

```python
response = client.recv(4096)
print(response.decode('utf-8'))
```

この場合、一度に受け取るデータサイズが4096バイト。

プリントするときはデコードする。

```python
client.close()
```

そしてソケットオブジェクトは最後にクローズ。

# UDPクライアント

あまりTCPクライアントと変わらない。

```python
import socket

HOST = '127.0.0.1'
PORT = 9997

client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
client.sendto(b'AAABBBCCC', (HOST, PORT))

data, address = client.recvfrom(4096)
print(data.decode('utf-8'))
print(address.decode('utf-8'))

client.close()
```

UDPクライアントゆえにソケットタイプが`SOCK_STREAMからSOCK_DGRAM`に変わった。

```python
client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
```

次にデータを送信する。

```python
client.sendto(b'AAABBBCCC', (HOST, PORT))
```

TCPクライアントの時と違い、UDPクライアントでは`sendto`メソッドでデータと（ホスト,ポート番号）のタプルを一度に渡してしまう。

周知の通り、UDPはコネクションレス型だから`client.connect`とか無い。

```python
data, address = client.recvfrom(4096)
```

UDPクライアントでは、`client.recvfrom`で「データ」と「リモートホストの情報」を一度にもらう。

# TCPサーバ

```python
import socket
import threading

IP = '0.0.0.0'
PORT = 9998


def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind((IP, PORT))
    server.listen(5)
    print(f'[*] Listening on {IP}:{PORT}')

    while True:
        client, address = server.accept()
        print(f'[*] Accepted connection from {address[0]}:{address[1]}')
        client_handler = threading.Thread(target=handle_client, args=(client,))
        client_handler.start()


def handle_client(client_socket):
    with client_socket as sock:
        request = sock.recv(1024)
        print(f'[*] Received: {request.decode("utf-8")}')
        sock.send(b'ACK')


if __name__ == '__main__':
    main()
```

まずはサーバーにリッスンして欲しいIPアドレスとポート番号を設定する。

```python
IP = '0.0.0.0'
PORT = 9998
server.bind((IP, PORT))
```

接続してきたクライアントソケットと、接続してきた相手の情報（IPアドレスとポート番号のタプル）の計二つを受け取る。

```python
client, address = server.accept()
```

threadingモジュールを使って（スレッドベースの）並列処理を実装する。

```python
client_handler = threading.Thread(target=handle_client, args=(client,))
```

これはスレッドを使用して関数`handle_client`を呼び出すということ。`args`でその関数に渡す引数を設定。

この引数clientは上でacceptした「接続してきたクライアントのソケットオブジェクト」だね。

スレッドの開始：

```python
client_handler.start()
```

pythonのthreadingについて：[[Python] スレッドで実装する](https://qiita.com/tchnkmr/items/b05f321fa315bbce4f77)


あとはクライアントからのデータを受け取って、まぁ一応の確認として”ACK”の文字をクライアントに送り返してやる。

```python
def handle_client(client_socket):
    with client_socket as sock:
        request = sock.recv(1024)
        print(f'[*] Received: {request.decode("utf-8")}')
        sock.send(b'ACK')
```
