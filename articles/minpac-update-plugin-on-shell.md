---
title: "シェルのコマンドラインから Vim のプラグインを更新する with minpac"
emoji: "✌️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim"]
published: true
published_at: 2023-12-07 00:00
publication_name: "vim_jp"
---

:::message
この記事は [Vim Advent Calendar 2023](https://qiita.com/advent-calendar/2023/vim) シリーズ1の 12/6 の記事です。
:::

:::message
この記事中のコードはとりあえず bash/zsh（私の普段使いのシェル） 向けのコードになっています。他の環境の方はイイカンジに読み替えてもらえるとありがたいです。
:::

# モチベーション

私は以下のような感じで、ソフトウェアを一括アップデートできるようなコマンドを用意して、適当にこのコマンドを叩くことでソフトウェアのアップデートをしています。

```bash
function update-softwares() {
    local password=''
	sudo -k  # Reset sudo credential cache
	echo -n 'Password:'; read -s password;
	while ! sudo -Svp '' &> /dev/null <<< $password; do
			echo
			echo 'Sorry, try again.'
			echo -n 'Password:'; read -s password;
	done

	if which brew &> /dev/null; then
		brew upgrade
		brew cleanup
		if [[ "$(uname)" == "Darwin" ]]; then
			brew upgrade --cask
		fi
	fi

	if which pacman &> /dev/null; then
		if which yay &> /dev/null; then
			yay -Syyu --noconfirm --sudoflags -S <<< $password
		else
			sudo -S pacman -Syyu --noconfirm <<< $password
		fi
	fi

	# and so on...
}
```

このコマンドは定期的に叩いているので、このコマンドを叩いたときに一緒に Vim のプラグインもアップデートできたら便利では？というのが今回の記事のネタのモチベーションです。

# 実装

Vim のプラグインを更新する独立したコマンドとして紹介します。
実際私も Vim のプラグインを更新するコマンドは独立したコマンドとして定義していて、このコマンドを先述した `update-softwares` コマンドの末尾で呼んでやることでソフトウェアの更新と同時に Vim のプラグインの更新もできるようにしています。


```bash:.bashrc やら .zshrc やらお好みのところにどうぞ
function update-vim-plugins() {
	echo 'Updating vim plugins'
	# 次の行の /path/to/your/.vimrc と /path/to/update_minpac.vim の部分は
	# 良い感じに書き換えてください。
	vim -u "/path/to/your/.vimrc" -i NONE --noplugins -n -N -e -s \
		-S "/path/to/update_minpac.vim"
}
```

```vim:update_minpac.vim
function UpdatePlugins()
  " PackInit() 関数は .vimrc に記述されている関数で、minpac をロードしてインス
  " トールしたいプラグインを登録する関数だとします。ご自身の環境に合わせていい
  " 感じに書き換えてください。既存の minpac ユーザーなら相当するものがあるはず
  " ですよね。
  call PackInit()

  " エラー処理のブロックです。何らかの理由で minpac のロードに失敗した時に実行
  " されます。:print コマンドはバッチモードにおいて例外的に出力メッセージを標
  " 準出力に流すので、これを用いて Vim のメッセージログを標準出力に流したの
  " ち、:cquit! で Vim を異常終了しています。
  if !exists("g:loaded_minpac")
    " :print コマンドに出力させるためのメッセージをバッファにセット
    call setline(1, split(execute("message"), "\n"))
    call append("$", "Failed to load minpac")
    " 標準出力に流す
    %print
    " Vim を終了
    cquit!
  endif

  " プラグインの更新情報を自動で出すようにします。
  let g:minpac#opt.status_auto = v:true

  " プラグインの更新を始めます。第2引数に渡した設定により、更新が終わったら
  " PostUpdatePlugins() 関数が自動で呼ばれるようにしています。
  call minpac#update("", {"do": "call PostUpdatePlugins()"})
endfunction

function PostUpdatePlugins()
  " minpac がプラグインの更新情報をカレントバッファに残しているので、それを
  " 標準出力に流したのち、Vim を正常終了させています。
  %print
  quitall!
endfunction

" 自動コマンドを用いて Vim の起動後に UpdatePlugins() 関数が呼ばれるようにして
" います。
autocmd VimEnter * ++once call UpdatePlugins()
```

# シェルスクリプト部解説

実は基本的には [thinca](https://zenn.dev/thinca) さんの「永遠に未完成」というブログの「[Vim script で AtCoder に参戦する方法](https://thinca.hatenablog.com/entry/20151003/1443853833)」というエントリーで紹介されている手法の応用です。感謝。
ちなみに余談ですが、今は AtCoder では公式に~~なぜか~~ Vim script ランナーが用意されているので、こんな細工をしなくても AtCoder に Vim script で参戦することができます。


あと Vim script 部解説についてですが、Vim script 部の解説はコメントとしてコードに埋め込んだ方がわかりやすいだろうと思ったのでコメントに埋め込みました。上記コードのコメントを参照ください。

では、本題に戻ってシェルスクリプト部解説です。まず 1 行目。

```bash
echo 'Updating vim plugins'
```

単に標準出力にメッセージを残しているだけです。特に特別なことはありません。


次に2行目。

```bash
vim -u "/path/to/your/.vimrc" -i NONE --noplugins -n -N -e -s \
	-S "/path/to/update_minpac.vim"
```

ここが `update-vim-plugins` コマンドの心臓部です。大雑把に一言でまとめるなら、Vim を headless なモードで起動して `update_minpac.vim` を実行する、ということをしています。それぞれのコマンドライン引数を解説すると以下のような感じです。
- `-u "/path/to/your/.vimrc"`
  起動時に読み込む vimrc を指定します。バッチモード(後述します)で起動した時は vimrc が自動で読み込まれないので、明示的にコマンドライン引数として与えています。vimrc で定義している(と仮定している) `PackInit()` 関数を Vim script 部で使えるようにするためです。
- `-i NONE`
  viminfo ファイルを使用しないようにします。
- `--noplugins`
  自動でプラグインを読み込まないようにします。Vim の起動時間の短縮のためです。
- `-n`
  スワップファイルを使用しないようにします。
- `-N`
  Vi 互換モードをオフにして、いわゆる Vim の機能がまるまる使える状態で起動するようにします。
- `-e -s`
  Vim をバッチモードで起動します。Vim の画面を出さずに Vim のコマンドを実行できるモードです。一部例外を除いて、エラーメッセージなどの各種メッセージなどが基本的に画面に出力されないようになります。`-e` と `-s` の両方セットで指定したときに有効になります。
- `-S "/path/to/update_minpac.vim"`
  Vim の起動後に与えられたファイル `"/path/to/update_minpac.vim"` を Vim script として実行します。

というわけでこれらをまとめると、2 行目は「Vim を余計なものを削ぎ落とした上でバッチモードで起動して、起動後に `update_minpac.vim` を実行するコマンド」ということになります。
上で紹介したコマンドライン引数などへのヘルプの一覧を用意しておきます。一次情報にあたりたい方は参考にしてください。

|コマンドライン引数など|ヘルプ|
|:--|:--|
|`-u "/path/to/your/.vimrc"`|`:h -u`|
|`-i`|`:h -i`|
|`--noplugins`|`:h --noplugins`|
|`-n`|`:h -n`|
|`-N`|`:h -N`|
|`-s`、バッチモード|`:h -s-ex`|
|`-S`|`:h -S`|


ちなみに完全に余談ですが、これらのフラグを使うと Vim script を使った実行スクリプトをかけたりします。

```bash
$ cat ./hello-by-vim
#!/usr/bin/env vim -u NONE -i NONE -n -N -e -s -S

call setline(1, 'hello')
%print
qa!
$ chmod u+x ./hello-by-vim
$ ./hello-by-vim
hello
```


# おわりに

システムの更新と一緒に勝手に Vim のプラグインの更新も走るというのはそれなりに便利です。他のパッケージマネージャでも応用できる話だと思うので、お気に召しましたらどうぞ。


# おまけ：Vim script 部をシェルスクリプト部に統合する

上で紹介した thinca さんの「[Vim script で AtCoder に参戦する方法](https://thinca.hatenablog.com/entry/20151003/1443853833)」の記事をご覧になった方にはすでにネタは割れていることとは思いますが、bash や zsh であれば、シェルのプロセス置換という機能を使って `update_minpac.vim` に記述した Vim script 部をシェルスクリプトにインラインで記述することができます。移植性は落ちますが、わざわざ別で Vim script のファイルを用意する必要がなくなって便利かもしれません。記事では、例えば Windows の cmd.exe や PowerShell 等の別環境でも移植しやすいように、ということで Vim script 部を別ファイルにして紹介しました。

```bash
function update-vim-plugins() {
	echo 'Updating vim plugins'
	vim -u "path/to/your/.vimrc" -i NONE --noplugins -n -N -e -s -S <(cat <<- EOF
	function UpdatePlugins()
		PackInit
		if !exists("*minpac#init()")
			call setline(1, split(execute("message"), "\n"))
			call append("$", "Failed to load minpac")
			%print
			cquit!
		endif
		let g:minpac#opt.status_auto = v:true
		call minpac#update("", {"do": "call PostUpdatePlugins()"})
	endfunction
	function PostUpdatePlugins()
		%print
		quitall!
	endfunction
	autocmd VimEnter * ++once call UpdatePlugins()
	EOF
	)
}
```
