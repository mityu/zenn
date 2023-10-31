---
title: ":terminal から親の Vim でファイルを開く(cmd.exe 編)"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim"]
published: true
publication_name: "vim_jp"
---

::: message
この記事は [Vim 駅伝](https://vim-jp.org/ekiden/) の 2023/10/30 の記事です。
前回の記事は atusy さんの「[テキストの折り畳みを彩る vim.treesitter.foldtext() を使ってみる](https://vim-jp.org/ekiden/#article-2023-10-27)」でした。
次回の記事は 11/1 に投稿される予定です。
:::

今回の記事は先日書いた 「[:terminal から親の Vim でファイルを開く(bash/zsh編)](https://zenn.dev/vim_jp/articles/5fdad17d336c6d)」 という記事の続編になります。まだそちらを読んでいない方は、まずそちらから読んでもらった方が良いかと思います。また前回の記事同様、今回の話も Vim 限定のお話で Neovim では使えませんがご容赦ください。

# コマンドプロンプトでも drop コマンド使いたくなるよね

前回の記事で :terminal で開いた bash や zsh から親の Vim でファイルを開く drop コマンドを実装しました。こうなると、Windows で Vim を使っているときにも :terminal で開いたコマンドプロンプトで drop コマンドを使いたくなるのが人情です。あれめっちゃ便利だし。というわけで次のように `drop.bat` を用意し、パスを通して試してみます。

```bat:drop.bat
@echo off
if "%VIM_TERMINAL%" == "" goto :EOF
powershell -Command "$ESC=[char]27;$BEL=[char]7;echo ""${ESC}]51;[`"""call`""", `"""Tapi_drop`""", [`"""%cwd%`""", `"""%1`"""]]${BEL}"""
```

cmd.exe の echo コマンドでは `<ESC>` 等が出力できないようなので、Terminal API を叩くのに PowerShell のワンライナーを使っています。なお、このワンライナーは vim-jp で教えていただきました。感謝。

さて、話を本筋に戻して、上のように `drop.bat` を用意して drop コマンドを叩いてみても、親の Vim では何もファイルが開かれません。これだけだとバッチファイルのバグを疑うこともできますが、試しに次のような Terminal API の drop コマンドを叩く C 言語のプログラムを実行してみてもファイルが開かれることがなかったので、バッチファイルが悪いのではなく、そもそも :terminal から Terminal API をうまく叩けていないようだ、となります。

```c:main.c
#include <stdio.h>

int main(void) {
    puts("\e]51;[\"drop\", \"fileA.txt\"]\x07");
}
```

```:コマンドプロンプト
> gcc main.c
> a.exe
?]51;["drop", "fileA.txt"]
```

# 何が悪いのか

上記の話を vim-jp でしてみたところ、次のような話になりました。Windows において、 Vim は :terminal で必要な仮想端末を作るのにデフォルトでは winpty というものを使っているのですが、これは少し古い仕組みで動いていて、そのせいでエスケープシーケンスをうまいこと扱えていないのではないか、ということのようです。細やかな検証をしたわけではないので、「どうも winpty が足を引っ張ってるっぽい」ぐらいの温度感でとどまっていて、もし冤罪だったらごめーん、という感じではあるのですが、とりあえず現状で動かないことには変わりないので何か別の解決策を探してみましょう。

# 解決策その1：ConPTY を使う

先の話と一緒に vim-jp ででた解決策です。ただ、後述しますがこれは少し問題があるので、私は次の解決策その2 を採用しています。


先ほど Vim は Windows で仮想端末を作るのに winpty を使うと述べましたが、実は Vim は winpty ではなく ConPTY というものをを使う実装も内蔵しています。どちらの実装を使うかは `'termwintype'` というオプションで設定でき、2023/10/30 現在では winpty がデフォルトになっています。ConPTY に問題があるため、こうなっているようです。[^1]
ConPTY は先述した `'termwintype'` オプションの値を `conpty` にするか、あるいは :terminal を開くときに

```
:terminal ++type=conpty
```

のようにして `++type=conpty` を指定することで明示的に使えるようになります。


試してみたところ、ConPTY だとエスケープシーケンスやらをうまいこと扱えるようで、Terminal API が正しく動作しましました。もし、ご自身の環境で ConPTY がいい感じに動作するのであれば採用しても良いかと思います。


ただ、この説の先頭でも書いた通り ConPTY には少し問題があり、これを使っていると Vim がハングすることがあるようです。というか実際私は gVim がハングして ConPTY を使うやり方を諦めました。
もしご自身で ConPTY を試してみる場合は、一応急に Vim が黙りこくって返事をしなくなっても大丈夫な状態で試してみることをお勧めします。私はファイルをこまめに保存していたおかげで手傷を負わずに済みました。あぶなかった。まあ趣味コードだったんで最悪消えてもなんとか...ではありましたが。


[^1]: `:h ConPTY` にこの記述があります。この記事を書いた時点での記述はここから見られます：https://github.com/vim/vim/blob/0ab500dede4edd8d5aee7ddc63444537be527871/runtime/doc/terminal.txt#L462C22-L468

# 解決策その2：client server の機能を使う

私が現在採用している方法です。


Terminal API とは全く別の機能の、client server という機能を用いて実現します。
client server とは、Vim をコマンドサーバーとして使えるようにする機能で、

```
vim --servername {サーバー名} --remote-send "{送信するキー}"
```

のようにして `vim` コマンドを叩くことで Vim の外部から Vim にコマンドを投げて実行させることができます。これを使うには Vim が `+clientserver` でコンパイルされている必要があり、`:echo has('clientserver')` としたときの出力が `1` なら有効、`0` なら無効、として判定できます。
client server についての詳細は `:h client-server` を参照してください。

とりあえずここで重要なのは、`vim` コマンドを叩くことで既に起動している Vim で何かしらのコマンドを実行できるという点です。つまり、client server で `:call Tapi_drop({イイ感じの引数})` を実行するコマンドを親の Vim に投げてやることで、Terminal API の call コマンドを無理やりエミュレーションしようということです。[^2]

[^2]: どうでも良い話ですが、ファイルパスを相対パスから絶対パスに変換する処理を Vim script 側で記述したのが結果的に良い方向にはたらいて良かったですね。ここでの実装が楽になりました。

Terminal API 経由で呼び出される関数は、第一引数に :terminal のバッファ番号、第二引数にリストでその関数で使う引数を渡すということになっています。そして `Tapi_drop` 関数はその第二引数のリストとしてカレントディレクトリと開くファイルのファイル名を指定することになっていました。なので、この関数の呼び出しは、この関数が :terminal のバッファ番号の情報は利用していないためバッファ番号としては適当に `0` を渡すことにして、`call Tapi_drop(0, ['%cd%', '%1'])` のようにすれば良いです。
また、キーシーケンスの送信先のサーバー名については、親の Vim が `VIM_SERVERNAME` という環境変数にサーバー名を設定してくれているので、これを用います。

というわけで用意する `drop.bat` は次のようにします。

```bat:drop.bat
@echo off
REM :terminal 内じゃなければスキップ
if "%VIM_TERMINAL%" == "" goto :EOF
vim --servername %VIM_SERVERNAME% --remote-send "<Cmd>call Tapi_drop(0, ['%cd%', '%1'])<CR>"
```

この `drop.bat` を PATH の通った場所においてやると、晴れて :terminal 上のコマンドプロンプトでも drop コマンドが動くようになります。

# おわりに

今回の記事は vim-jp におんぶにだっこでお送りしました。集合知というのは素晴らしいですね。
