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

そして、今回は





