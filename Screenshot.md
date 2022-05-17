# get_dimensions関数について

```python
def get_dimensions():
    width = win32api.GetSystemMetrics(win32con.SM_CXVIRTUALSCREEN)
    height = win3api.GetSystemMetrics(win32con.SM_CYVIRTUALSCREEN)
    left = win3api.GetSystemMetrics(win32con.SM_XVIRTUALSCREEN)
    top = win32api.GetSystemMetrics(win32con.SM_YVIRTUALSCREEN)
    return (width, height, left, top)
```

`GetSystemMetrics`は、さまざまなシステムメトリックの値（表示要素の幅と高さ）とシステムの現在の構成を取得する。

表示要素とは、ウィンドウの一部、またはシステムが表示する画面の一部のこと。

その引数に与えてるやつらはこれ（vs=バーチャルスクリーン）：

- `SM_CXVIRTUALSCREEN` → vsの幅
- `SM_CYVIRTUALSCREEN` → vsの高さ
- `SM_XVIRTUALSCREEN`→ vs左側の座標
- `SM_YVIRTUALSCREEN` → vsのトップの座標

> バーチャルスクリーンとは、実際に表示可能な解像度よりも大きい解像度を使用できるようにするための仮想画面のこと。
> バーチャルスクリーンでは、仮想画面全体のサイズが実際のディスプレイの画面より大きくなるから、ディスプレイには常に仮想画面の一部が表示されている状態となる。
> 画面をスクロールすると、ディスプレイの画面上で見えていない部分を表示することができる。

# screenshot間数について

（画面全体のスクリーンショットを撮ることを目的とする。）

```python
def screenshot(name='screenshot'):
    hdesktop = win32gui.GetDesktopWindow()
    width, height, left, top = get_dimensions()

    desktop_dc = win32gui.GetWindowDC(hdesktop)
    img_dc = win32ui.CreateDCFromHandle(desktop_dc)
    mem_dc = img_dc.CreateCompatibleDC()

    screenshot = win32ui.CreateBitmap()
    screenshot.CreateCompatibleBitmap(img_dc, width, height)
    mem_dc.SelectObject(screenshot)
    mem_dc.BitBlt((0,0), (width, height), img_dc, (left, top), win32con.SRCCOPY)
    screenshot.SaveBitmapFile(mem_dc, f'{name}.bmp')

    mem_dc.DeleteDC()
    win32gui.DeleteObject(screenshot.GetHandle())
```

（途中で「今どこやってんだ？」って迷子になりかねないからはじめに全体像を示した。）

それぞれの説明があるから部分ごとにみていく。

```python
hdesktop = win32gui.GetDesktopWindow()
width, height, left, top = get_dimensions()
```

まず`GetDesktopWindow()`でデスクトップウィンドウのハンドルを取得。

（デスクトップウィンドウはスクリーン全体を覆っており、この上にアイコンやウィンドウなどが描画される。）

```python
desktop_dc = win32gui.GetWindowsDC(hdesktop)
img_dc = win32ui.CreateDCFromHandle(desktop_dc)
```

`GetWindowsDC`でタイトルバー、メニュー、スクロールバーを含む、ウィンドウ全体のデバイスコンテキスト（DC）を取得。

引数は、DCを取得したいウィンドウのハンドルを指定する。

> デバイスコンテキスト（DC）とは、ディスプレイやプリンターなどのデバイスの描画属性に関する情報を含むWindows データ構造。

`img_dc`についてなんだけど、[公式ドキュメント](http://timgolden.me.uk/pywin32-docs/win32ui__CreateDCFromHandle_meth.html)には“Creates a DC object from an integer handle.”としか書いてない。誤植？

とにかくDCオブジェクト。

```python
mem_dc = img_dc.CreateCompatibleDC()
```

メモリDVを作成（メモリ内に作成される）。

ビットマップをファイルに書き込むまで、ここにキャプチャした画像を保管しておく。

描画処理をする前に、メモリDCに幅と高さを`CreateCompatibleBitmap`関数で指定し、ビットマップ オブジェクトを作成する必要がある。

```python
screenshot = win32ui.CreateBitmap()
screenshot.CreateCompatibleBitmap(img_dc, width, height)
mem_dc.SelectObject(screenshot)
```

キャプチャしたデスクトップのDVと互換性のあるビットマップオブジェクト`screenshot`がこれでできる（最初の２行）。

その後、メモリDVとビットマップを関連付ける（`SelectObject`）。

```python
mem_dc.BitBlt((0, 0), (width, height),
                    img_dc, (left, top), win32con.SRCCOPY)
screenshot.SaveBitmapFile(mem_dc, f'{name}.bmp')
```

ビットブロック転送を行う。コピー元からコピー先のデバイスコンテキストへ、指定された長方形内の各ピクセルの色データをコピーする。

つまりメモリーデバイスコンテキストにデスクトップ画面をコピー。

`BitBlt`の詳しい引数は[こちら](https://www.tokovalue.jp/function/BitBlt.htm)。

`SaveBitmapFile()`で画像をファイルに書き起こす。

# 参考にしたページ

情報欲しい人は以下の参考リンクあたってみて：

- [Windows APIについて](https://www.tokovalue.jp/)
- [バーチャルスクリーンについて](https://support.nec-lavie.jp/e-manual/m/nx/vp/201110/pdf/pg/sw6/v1/mst/versapro_wxp_manual/_manual/02_kakubu/02-11_disp/VV51021106.htm)
- [GetSystemMetrics function (winuser.h)](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getsystemmetrics)
- [GetDesktopWindow(デスクトップウィンドウのハンドルを取得)](http://csharp.maeyan.net/Blog/Details/39)
- [【Win32】CreateCompatibleDC 関数](https://ameblo.jp/sgl00044/entry-12513832702.html)
- [Module win32ui](http://timgolden.me.uk/pywin32-docs/win32ui.html)
- [Module win32gui](http://timgolden.me.uk/pywin32-docs/win32gui.html)























