---
title: "Zsh on :terminal on Vim でカレントディレクトリを Vim と同期する"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim"]
published: true
published_at: 2023-05-31 00:00
publication_name: "vim_jp"
---

:::message
この記事は [Vim 駅伝](https://vim-jp.org/ekiden/) の 2023/5/31 の記事です。
前回の記事は [uga-rosa](https://zenn.dev/uga_rosa) さんの「[ddu.vimにちゃんと入門した話](https://zenn.dev/uga_rosa/articles/f55be75a573c0b)」という記事でした。
次回の記事は staticWagomU さんの [キーマッピングを設定しないVimmerが居たっていい](https://wagomu.me/blog/2023-06-02-vim-ekiden) です。
:::

### デモ

![](/images/sync-cwd-zsh-on-vim-demo.gif)

### なにこれ

`synccwd` という、`:terminal` で実行している zsh 上で、カレントディレクトリをVim のカレントディレクトリに変更するコマンドです。
Vim に固有の `terminal-api` という機能を用いて実現しているため、残念ながら Neovim 上では動きません。ご容赦。
また、zsh のローカル変数を宣言する機能を使っているので、この記事のコードのコピペだとシェルも zsh のみでしか動作しないと思いますが、それさえ解決すれば bash とかでも動くようになるんじゃないかと思います。

### 設定方法

.zshrc、.vimrc にそれぞれ以下のコードを記述するだけです。
コードの解説はコード中にコメントで記してあるので、そちらを参考にしてください。

```sh:.zshrc
# $VIM_TERMINAL 変数は、Vim が :terminal でシェルを開いた時に設定する環境変数です。
# synccwd コマンドは :terminal 上にいる時のみ定義したいので、この環境変数をチェッ
# クして、空でない時のみ synccwd コマンドを定義するようにします。
if [ -n "$VIM_TERMINAL" ]; then
	function synccwd() {
		local cwd

		# Vim の terminal-api という機能を利用して、Vim のユーザー定義関数
		# "Tapi_getcwd" 関数を呼び出しています。terminal-api の詳細については
		# :h terminal-api を参照してください。
		echo "\e]51;[\"call\", \"Tapi_getcwd\", []]\x07"

		# 上で呼び出した Tapi_getcwd 関数が Vim のカレントディレクトリのパスを
		# 標準入力に書き込んでくれるので、それを待機。書き込まれたパスを $cwd
		# 変数に保存する。
		read cwd

		# cd。
		cd "$cwd"
	}
fi

```

```vim:.vimrc
" 第一引数の `bufnr` には、シェルを開いている :terminal のバッファ番号が入っている。
function! Tapi_getcwd(bufnr, ...) abort
  " Vim のカレントディレクトリの取得。
  let cwd = call('getcwd', a:000)

  " `bufnr` で示される :terminal のバッファに紐づいているチャネルを取得。
  let channel = term_getjob(a:bufnr)->job_getchannel()

  " 取得したチャネルを使って、標準入力に Vim のカレントディレクトリのパスを書き込み。
  " ここで書き込んだデータが `synccwd` コマンド内の `read cwd` コマンドで読み出される。
  call ch_sendraw(channel, cwd . "\n")
endfunction
```

### おわりに

zsh のカレントディレクトリを Vim のカレントディレクトリと揃える `synccwd` コマンドの紹介でした。
私は正直使用頻度はあまり高くなく、たまーにあると便利かな？という感じなのですが、誰かこのコマンドが刺さる人がいれば幸いです。

ちなみに、これの逆のこと、すなわち `:terminal` で開いてるシェルのカレントディレクトリを変更したらVim のカレントディレクトリもそれに追従して変更されるようにする、ということは tyru さんの `sync-term-cwd.vim` というプラグインで実現可能です。よければチェックしてみてください。

- https://github.com/tyru/sync-term-cwd.vim


### 余談

Vim script に詳しい方とかだと気づかれてると思うのですが、`synccwd` 関数内の `Tapi_getcwd` 関数の呼び出しを

```sh
# ウインドウ番号 3 のカレントディレクトリに揃える
echo "\e]51;[\"call\", \"Tapi_getcwd\", [3]]\x07"
```

だとか


```sh
# タブページ番号 1、ウインドウ番号 3 のカレントディレクトリに揃える
echo "\e]51;[\"call\", \"Tapi_getcwd\", [3, 1]]\x07"
```

みたいな感じにすることで、一応特定のウインドウのカレントディレクトリを参照できるようになってます。（Vim の `getcwd()` 関数に渡す引数をシェル側から指定できるようなコードになってます。）
ただ、今のところ私は使い途が全然わからないので余談という形で紹介することにしました。もし使い途を見出した人がいらっしゃればどうぞ...。

......使い途...🤔

