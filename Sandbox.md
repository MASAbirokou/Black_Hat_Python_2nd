# サンドボックスとは？

> サンドボックスは、隔離された領域でプログラムを実行し、問題発生時においてもほかのプログラムに影響を及ぼさないようにする仕組みのことだ。和訳では公園の「砂場」を意味し、
> 外部から仕切られた環境で自由に遊べる状況に由来する。

普通の人間がパソコンを操作していれば、マウスの動きや何かしらの入力がある。

しかしサンドボックス内では、大抵マルウェアを自動解析しているため人間的なインタラクションが無い。

今回はそれを利用して、我々のトロイの木馬自身がが今自分はサンドボックス内にいるのか？を判断するという内容。

# get_last_input()関数

```python
class LASTINPUTINFO(Structure):
    fields_ = [
        ('cbSize', c_uint),
        ('dwTime', c_ulong)
    ]
```

クラス`LASTINPUTINFO`は、最新の入力タイムスタンプを保持する。

クラス変数に二つのタプルを要素とするリスト（構造体）。

`cbSize`がunsigned int型で構造体のサイズを表す。`dwTime`がunsigned long型で、入力の時間・タイミング。

```python
def get_last_inut():    
    struct_lastinputinfo = LASTINPUTINFO()
    struct_lastinputinfo.cbSize = sizeof(LASTINPUTINFO)
    
    windll.user32.GetLastInputInfo(byref(struct_lastinputinfo))    
    run_time = windll.kernel32.GetTickCount()

    elapsed = run_time - struct_lastinputinfo.dwTime
    print(f"[*] It's been {elapsed} milliseconds since the last event.")
    return elapsed
```

この関数は最後の入力からの経過時間（=elapsed）を返す。

まずインスタンス`struct_lastinputinfo`を作って、そのサイズを指定。

ctypesモジュールの`sizeof`は、ctypesの型やインスタンスのメモリバッファのサイズをバイト数で返す。（Cのsizeof演算子と同じ）

`GetLastInputInfo`で、最後の入力イベントの時刻が入る。

引数は`LASTINPUTINFO`構造体へのポインタ（`byref`は指定した引数へのポインタ）。

`GetTickCount`はシステムを起動した後の経過時間がミリ秒単位で返る。

ゆえに経過時間はあのような引き算になる。

# Detectorクラスのメソッド達

初期化メソッドは以下の通り：

```python
def __init__(self):
        self.double_clicks = 0
        self.keystrokes = 0
        self.mouse_clicks = 0
```

# get_key_pressメソッド

```python
def get_key_press(self):
        for i in range(0, 0xff):
            state = win32api.GetAsyncKeyState(i)
            if state & 0x0001:
                if i == 0x1:
                    self.mouse_clicks += 1
                    return time.time()
                elif i > 32 and i < 127:
                    self.keystrokes += 1
        return None
```

`get_key_press()`は、インスタンス変数を変化させる動きをする。

返り値は、ある時は（後々計算で使うように）UNIXエポックからの経過秒数を返す（time.time()）。

> UNIX時間とは、[UTC（協定世界時）](https://ja.wikipedia.org/wiki/%E5%8D%94%E5%AE%9A%E4%B8%96%E7%95%8C%E6%99%82)の1970年1月1日00:00:00秒（＝「UNIXエポック：
> UNIX Epoch」と呼ばれる）からの経過秒数のことである。

`GetAsyncKeyState`関数は、関数呼び出し時にキーが押されているかどうか、また前回の`GetAsyncKeyState`関数呼び出し以降にキーが押されたかどうかを判定する。

引数は仮想キーコード。

キーコード0x01はマウスの左ボタン。32から127は一般的なキー（詳しくは[ここ](https://kts.sakaiweb.com/virtualkeycodes.html)）。

関数が成功すると、前回の`GetAsyncKeyState`関数呼び出し以降にキーが押されたかどうか、およびキーが現在押されているかどうかを示す値が返る。

最上位ビット（0x8000)がセットされたときは現在そのキーが押されていることを示し、最下位ビット(0x0001)がセットされたときは前回の`GetAsyncKeyState`関数呼び出し以降にそのキーが
押されたことを示す。

# detectorメソッド

長いから、detectorメソッドの前半・後半を分けて示す。

（detectorメソッドの前半部分）：

```python
def detector(self):
        previous_timestamp = None
        first_double_click = None
        double_click_threshold = 0.35
        
        max_double_clicks = 10
        max_keystrokes = random.randint(10,25)
        max_mouse_clicks = random.randint(5,25)
        max_input_threshold = 30000

        last_input = get_last_input()
        if last_input >= max_input_threshold:
            sys.exit(0)
```

はじめで定義してる各種変数は頭の片隅に入れておく。

ちなみに`random.randint(a, b)`で`a`以上`b`以下の整数をランダムに返す（すなわち端っこの数`b`も入る）。

`last_inputh`は`get_last_inut`関数の返り値、つまり経過時間。

だから、その経過時間が不自然に長かったらサンドボックス内にいるということ。そして何事もなかったかのように`sys.exit(0)`で正常終了。

（detectorメソッドの後半部分）：

```python
detection_complete = False
while not detection_complete:
    keypress_time = self.get_key_press()
    if keypress_time is not None and previous_timestamp is not None:
        elapsed = keypress_time - previous_timestamp
        
        if elapsed <= double_click_threshold:
            self.mouse_clicks -= 2
            self.double_clicks += 1
            if first_double_click is None:
                first_double_click = time.time()
            else:
                if self.double_clicks >= max_double_clicks:
                    if (keypress_time - first_double_click <= (max_double_clicks*double_click_threshold)):
                          sys.exit(0)
        if (self.keystrokes >= max_keystrokes and 
            self.double_clicks >= max_double_clicks and 
            self.mouse_clicks >= max_mouse_clicks):
            detection_complete = True
            
        previous_timestamp = keypress_time
    elif keypress_time is not None:
        previous_timestamp = keypress_time 
```

whileループが終わるのは、すなわち`detection_complete = True`となるのは、インスタンス変数`self.keystrokes`、`self.double_clicks`、`self.mouse_clicks`の値たちが
それぞれ`max_mouse_clicks`、`max_keystrokes`、`max_double_clicks`以上になった時。

まずキーの入力またはマウスのクリックがあったらタイムスタンプを取得（`keypress_time`）。

経過時間（`elapsed`）は前の入力時間との差。

`previous_timestamp`が`None`、つまりその時の入力が一番最初の入力だったら`previous_timestamp`に今のタイミングでの`keypress_time`を代入する。

（それ以降の`previous_timestamp`はその都度更新されていく。）

`double_click_threshold`はダブルクリックの際のクリックする間隔。

だから`elapsed`が`double_click_threshold`以下だったら、普通のクリックではなくダブルクリックだから、インスタンス変数`self.mouse_clicks`と`self.double_clicks`を調整。

それで、`first_double_click`はUNIXエポックからの経過秒数。

ここがポイント👇

```python
if self.double_clicks >= max_double_clicks:
    if (keypress_time - first_double_click <= (max_double_clicks * double_click_threshold)):        
        sys.exit(0)
```

ダブルクリックが多すぎる！

これはホワイトワッカーがこっちのテクニックを見越してダブルクリックしまくってる可能性がある。

へっ😏！！

裏の裏をかいてこっちもその対策だ！

何事もなかったかのように退散する。

結局、メソッド`detector()`は、`sys.exit(0)`するかしないかの判断をしてくれるということ。
