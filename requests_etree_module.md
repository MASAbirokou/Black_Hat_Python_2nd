# pythonのrequests,etreeモジュールについて

Black Hat Python【2nd Edition】のchapter5の補遺。

まず以下の通りimportする。

```python
import requests
from lxml import etree
from io import BytesIO
```

`requests.get`でURLを指定してHTTPリクエスト:
```python
r = requests.get('http://192.168.2.115')
```

今回は自分がVirtualBoxで立てたwebサーバーを対象にする。

この`requests.get`の返り値を公式ドキュメントにならって**リスポンスオブジェクト**と呼称する。

（responseの発音記号は[rɪspáns]なので”レス”ではなく”リス”とする。）

リスポンスオブジェクトの属性一覧は次の通り：

```
>>> print(dir(r))
['__attrs__', '__bool__', '__class__', '__delattr__', '__dict__', '__dir__',
'__doc__', '__enter__', '__eq__', '__exit__', '__format__', '__ge__', '__getat
tribute__', '__getstate__', '__gt__', '__hash__', '__init__', '__init_subclass
__', '__iter__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__non
zero__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__setstate
__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_content',
'_content_consumed', '_next', 'apparent_encoding', 'close', 'connection', 'con
tent', 'cookies', 'elapsed', 'encoding', 'headers', 'history', 'is_permanent_r
edirect', 'is_redirect', 'iter_content', 'iter_lines', 'json', 'links', 'nex
t', 'ok', 'raise_for_status', 'raw', 'reason', 'request', 'status_code', 'tex
t', 'url']
```

それぞ見てみると…

```
>>> r.reason
'OK'
>>> r.status_code
200
>>> r.url
'http://192.168.2.115/'
```

`r.text`はリクエストしたページのhtml。

str型であり、`print`で表示すれば`\n`等が解釈されて綺麗なhtmlコードが見れる。

それと違って、`r.content`はbytes型。

でも表示した結果はほぼ同じ：

```
b'<!DOCTYPE html>\n\n<html class="no-js" lang="en-US">\n\n\
t<head>\n\n\t\t<meta charset="UTF-8">\n\t\t<meta name="view
port" conte
...
<div style="clear: both;"> <a href="https://www.t
urnkeylinux.org/wordpress">WordPress Appliance</a> - Powered by <a href=
"https://www.turnkeylinux.org">TurnKey Linux</a> </div> </div
></body>\n</html&gt\n'
```

公式ドキュメント：[Requests: 人間のためのHTTP](https://requests-docs-ja.readthedocs.io/en/latest/)

# HTMLParserとparse

```
>>> r = requests.get('http://192.168.2.115/wp-login.php')
>>> content = r.content
>>> parser = etree.HTMLParser()
```

上記のようにリスポンスオブジェクトからの`content`とさらにパーサを用意する。

> 【DOMツリー（DOM Tree）】XML文書は、論理的には木構造となっている。最初に出現するルート要素は根（ルート）であると考えられ、すべての要素や属性は、そこから延びる枝葉として考えられる。
> DOMにおいては、読み込んだXML文書を実際に木（ツリー）の形をしたデータに変換する。このデータを、DOMツリーと呼ぶ。以下のようなXML文書は、下図のようなツリーに展開される。
> ![domtree](https://user-images.githubusercontent.com/85237728/168716370-7e1b0774-dbe1-4db7-adab-38dd9e4e6a0b.jpeg)

`etree.parse`は、与えたオブジェクトをツリー構造にパースしてくれる。

```python
content = etree.parse(BytesIO(content), parser = parser)
```

その時`etree.parse`に与えてよい対象は

- a file name/path
- a file object
- a file-like object
- a URL using the HTTP or FTP protocol

のようにファイル系オブジェクト。

 

しかし先に述べたように`content`すなわち`r.content`はbytes型。

それをファイル系オブジェクに変換してくれるのがBytesIO。

# ElementTreeオブジェクトのメソッドfindall


```
>>> content = requests.get('http://192.168.2.115/wp-login.php').content
>>> content = etree.parse(BytesIO(content), parser = parser)
>>> type(content)
lxml.etree._ElementTree
```

`etree.parse`の返り値はElementTreeオブジェクト。

属性を見てみると

```
>>> print(dir(content))
['__class__', '__copy__', '__deepcopy__', '__delattr__', '__dir__', '__doc__',
'__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '_
_init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__pyx
_vtable__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeo
f__', '__str__', '__subclasshook__', '_setroot', 'docinfo', 'find', 'findall',
'findtext', 'getelementpath', 'getiterator', 'getpath', 'getroot', 'iter', 'it
erfind', 'parse', 'parser', 'relaxng', 'write', 'write_c14n', 'xinclude', 'xml
schema', 'xpath', 'xslt']
```

# findallについて

説明するよりもコードを多めに示す。

ただし以降で出てくる`content`は上で得た`etree.parse`の返り値のこと。


```
>>> f = content.findall('//input')
>>> type(f)
list
>>> f
[<Element input at 0x7ff51c034ec0>,
 <Element input at 0x7ff51c034f00>,
 <Element input at 0x7ff51c034f40>,
 <Element input at 0x7ff51c034f80>,
 <Element input at 0x7ff51c034fc0>,
 <Element input at 0x7ff51c02f680>
 ]
>>> type(f[0])
<class 'lxml.etree._Element'>
```

findallの引数に`'//input'`を渡してinputタグの抽出を試みる。

```
>>> for link in content.findall('//input'):
...     print("link.get('name'): ", link.get('name'))
...     print("link.get('value'): ", link.get('value'))
...     print('')
...
link.get('name'):  log
link.get('value'):  

link.get('name'):  pwd
link.get('value'):  

link.get('name'):  rememberme
link.get('value'):  forever

link.get('name'):  wp-submit
link.get('value'):  ログイン
```

この結果とリクエスト先のhtmlコードの対応に注目！
（関係ない属性等は見やすさの為省いてる）

```
<input name="log" id="user_login" value=""/>
<input type="submit" name="wp-submit" value="ログイン" />
<input name="rememberme" value="forever"  />
<input type="submit" name="wp-submit" class="button button-primary button-large" value="ログイン" />
```

次は`findall`に`'//a'`を渡してaタグを抽出。

```
>>> for link in content.findall('//a'):
...     print("link.get('href'): ", link.get('href'))
...     print('link.text: ', link.text)
link.get('href'):  https://ja.wordpress.org/
link.text:  Powered by WordPress
link.get('href'):  https://192.168.2.115.com/wp-login.php?action=lostpassword
link.text:  パスワードをお忘れですか ?
```

htmlコードとの対応に注目！

```
<a href="https://ja.wordpress.org/">Powered by WordPress</a>
<a href="https://192.168.2.115.com/wp-login.php?action=lostpassword">パスワードをお忘れですか ?</a>
```

詳細はこちら：[xml.etree.ElementTree --- ElementTree XML API](https://docs.python.org/ja/3/library/xml.etree.elementtree.html#module-xml.etree.ElementTree)































































