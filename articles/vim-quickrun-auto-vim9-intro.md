---
title: ":QuickRun で Vim9 script の実行を簡単にするプラグイン作った"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim"]
published: true
publication_name: "vim_jp"
---

::: message
この記事は [Vim 駅伝](https://vim-jp.org/ekiden/) の 2025/2/24 の記事です。
前回の記事は [kamecha](https://zenn.dev/kamecha) さんの「[VimConfに行ってきた](https://trap.jp/post/2406/)」でした。
次回の記事は、[輪ごむ](https://zenn.dev/wagomu) さんの「[今夜はminiとsnacksで決まり！](https://wagomu.me/blog/2025-02-26-vim-ekiden)」です。
:::

# 導入

quickrun.vim (https://github.com/thinca/vim-quickrun) は、[thinca](https://zenn.dev/thinca) さん作のプラグインで、バッファに書かれたプログラムの全体や一部を簡単に実行できるプラグインです。
そして当然(？) Vim script や Vim9 script も実行できるのですが、Vim9 script 絡みでちょっと問題があり、Vim9 script で書いたスクリプトの一部を範囲選択して実行しようとすると、ちょっと面倒な場合があります。
例えば、以下のプログラムで `def F()` 〜 `F()` の行を範囲選択して quickrun で実行しようとすると、
`vim9script` コマンドが選択部分に含まれなくなるため、スクリプトが legacy Vim script だと認識されてエラーになってしまいます:

```vim
vim9script

# long long program...

def F()
  echo 'Hello'
enddef

F()  # この行は関数呼び出しコマンドの `:call` が省かれているので、Vim script だと実行できない
```

`:5,9 QuickRun`

```
function quickrun#command#execute[10]..quickrun#run[10]..112[10]..107[3]..<SNR>188_execute[9]..script /private/var/folders/91/jh__7nbx5wz9m_07frn6wn3h0000gn/T/vInYYEi/1, line 5
E464: Ambiguous use of user-defined command: F()
```

エラーを出さずに正しく実行するには、`long long program...` となっている部分を全部一時的にコメントアウトするか、あるいは以下のように、一時的に実行したいコードブロックの先頭に `vim9script` コマンドを挿入する必要があります。

```diff
vim9script

# long long program...

+vim9script
def F()
  echo 'Hello'
enddef

F()
```

こうすればプレーンな quickrun で Vim9 script のコード片を実行できますが、ただ、どちらにせよ少し手間がかかります。

# というわけで

quickrun-hook-auto_run_in_vim9 という quickrun のプラグインを作りました。(https://github.com/mityu/vim-quickrun-hook-auto_run_in_vim9)
このプラグインを使うと上述したような細工をせずに、ダイレクトに Vim9 script のコード片を実行できるようになります。
具体的には、vimrc などで以下のようなセットアップをした上で

```vim:vimrc
if !exists('g:quickrun_config')
  let g:quickrun_config = {}
endif
let g:quickrun_config.vim = {'hook/auto_run_in_vim9/enable': 1}
```

前節と同じように `:5,9 QuickRun` を実行すると、今度はちゃんと選択範囲が Vim9 script として実行されて、以下のように結果が得られます。

```
Hello
```

正直これが刺さる人はそんなに多くはないと思うのですが、刺さる人には便利に使ってもらえるものだと思います。最悪刺さる人がいなくとも、少なくとも自分は助かっているのでヨシ！

# 少しだけ内部の話

せっかくなので、内部実装の話を少しだけしておこうかと思います。

quickrun-hook-auto_run_in_vim9 は quickrun の hook 機能を使ってスクリプトの実行前に hook して

1. 実行する範囲の Vim script が legacy Vim script なのか Vim9 script なのかを判定する
1. 実行するスクリプトが Vim9 script であれば、スクリプトの先頭に `vim9script` コマンドが存在するかを調べる
1. 実行するスクリプトの先頭に `vim9script` コマンドがなければ、実行するスクリプトを書き換えて `vim9script` コマンドを補う

ということを行っています。
このプラグインでは、1. の部分の、実行するコード片が Vim9 script かどうかを判定する部分がキモになるのですが、この部分は昔私が作った vital-vim9context という vital モジュール[^vital.vim]を利用しています (https://github.com/mityu/vital-vim9context)。vital-vim9context を使うと、Vim script 中の任意の箇所が legacy Vim script なのか Vim9 script なのかを判定することができるので、このプラグインを作るのにはうってつけでした。このプラグインでは、実行されるコード片の先頭の位置を指定して、その部分が legacy Vim script なのか Vim9 script なのかを判定させています。

[^vital.vim]: vital.vim というのは、Vim script 向けのライブラリ群 (https://github.com/vim-jp/vital.vim) です。vital-vim9context は、この vital.vim 経由で扱えるライブラリ (vital モジュール) になります。

# 最後に

quickrun はいいぞ
