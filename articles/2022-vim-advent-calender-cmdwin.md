---
title: "Command-line window がちょっと便利になるかもしれない設定集"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "neovim"]
published: true
---

:::message
この記事は [Vim Advent Calender 2022（その２）](https://qiita.com/advent-calendar/2022/vim)の23日目の記事です。
:::

### はじめに

Command-line window とはなんでしょう？
普段これを使っていない方や、そもそもこんな機能知らなかったよ！という人向けには、ウインドウを閉じようと `:q` をタイプしようとして間違って `q:` とタイプしたときに出てくるウインドウ、となりますかね。cmdwin とも呼ばれたりすることもあるウインドウで、普通のバッファ上でコマンドなどを編集・実行できるウインドウです。

cmdwin のもうちょっと踏み込んだ導入については、[ちょうど今年の Vim Advent Calender 2022 に Command-line window についての記事がある](https://qiita.com/kaitat/items/4083003370a283bd2605)ので、そちらに譲りたいと思います。`:` で常に Command-line window を開くようにする設定なども紹介されているので是非どうぞ。またあるいは、「help だ、Vim の help を寄越せ！」という方は `:h cmdwin` を参照してください。

では、この記事では、Command-line window 生活がより便利になるかもしれない設定（小ネタという方が良いかもしれない）をいくつか紹介したいと思います。

:::message
~~ぶっちゃけタイピングがだるいのと~~字面的に少々間延びしてしまうので、以下では Command-line window を cmdwin と表記します。
:::


### 開かれる cmdwin の高さを変える

`'cmdwinheight'` というオプションを使って、開かれる cmdwin の高さを変更することができます。デフォルトでは 7 です。デフォルトで開かれる cmdwin の高さがちょっと高いなーとかちょっと低いなーとか思う場合はこれを変更してみると良いです。ちなみに私は 10 にしています。なんとなくです。

```vim
set cmdwinheight=10  " お好みの高さをどうぞ
```

### cmdwin 用の設定を書く

cmdwin の扱いをより良くしたい、となったときに、cmdwin 特有の設定を何か設定したくなることがあると思います。こういう設定を行うには、`CmdwinEnter` という自動コマンドで hook してあげると良いです。例えば、こんな感じのものを .vimrc に用意して、`" ここに cmdwin 用の設定を書いていく ...` のところに設定を追記していくようにすると良いでしょう。

```vim:.vimrc
augroup vimrc-cmdwin
  autocmd!
  autocmd CmdwinEnter * call s:setup_cmdwin()
augroup END

function! s:setup_cmdwin()
  " ここに cmdwin 用の設定を書いていく ...
endfunction
```

また、cmdwin 全般ではなく、例えばコマンド履歴の cmdwin を開いてる時だけなど、特定の cmdwin を開いた時のためだけの設定を書きたい場合は

 - `CmdwinEnter` 自動コマンドのパターンのところにコマンドラインの種類を表す文字を指定する
 - `getcmdwintype()` を使って現在の cmdwin の種類を表す文字を取得する
 - （`CmdwinEnter` 自動コマンドで呼ばれた時のみ）`expand('<afile>')` で現在の cmdwin の種類を表す文字を取得する

といった方法が使えます。なお、コマンドライン(cmdwin)の種類を表す文字というのは以下のようなものです。

|文字|コマンドラインの種類|
|:--:|:---|
|`:`|普通のExコマンド|
|`>`|デバッグモードのコマンド|
|`/`|前方検索に使われる文字列|
|`?`|後方検索に使われる文字列|
|`=`|Expression レジスタ `"=` 用の式|
|`@`|関数 `input()` に対して入力する文字列|
|`-`|コマンド `:insert` や `:append` に対して入力する文字列|

（参照: `:h cmdwin-char` ）
よく使うのは `:`（コマンド履歴） と `/`（前方検索履歴） と `?`（後方検索履歴）ですかね。

**例：**

※ここの例においては `augroup` の指定などは非本質的な部分になるので省略します。


 - `CmdwinEnter` を使った分岐

```vim
" コマンド履歴を開いたときだけ <C-n> でコマンド補完できるようにする
autocmd CmdwinEnter : inoremap <buffer> <expr> <C-n> pumvisible() ? '<C-n>' : '<C-x><C-v>'

" 前方検索/後方検索の履歴を開いたときだけ 'hlsearch' オプションをオンにする
autocmd CmdwinEnter [/\?] set hlsearch
```

 - `getcmdwintype()` を使った分岐

```vim
autocmd CmdwinEnter * call s:setup_cmdwin()

function! s:setup_cmdwin()
  " コマンド履歴を開いたときだけ <C-n> でコマンド補完できるようにする
  if getcmdwintype() ==# ':'
    inoremap <buffer> <expr> <C-n>
          \ pumvisible() ? '<C-n>' : '<C-x><C-v>'
  endif

  " 前方検索/後方検索の履歴を開いたときだけ 'hlsearch' オプションをオンにする
  if getcmdwintype() =~# '[/?]'
    set hlsearch
  endif
endfunction
```

 - `expand('<afile>')` を使った分岐

```vim
autocmd CmdwinEnter * call s:setup_cmdwin()

function! s:setup_cmdwin()
  " コマンド履歴を開いたときだけ <C-n> でコマンド補完できるようにする
  if expand('<afile>') ==# ':'
    inoremap <buffer> <expr> <C-n>
          \ pumvisible() ? '<C-n>' : '<C-x><C-v>'
  endif

  " 前方検索/後方検索の履歴を開いたときだけ 'hlsearch' オプションをオンにする
  if expand('<afile>') =~# '[/?]'
    set hlsearch
  endif
endfunction
```

### cmdwin を閉じるマッピングを用意する

何かコマンド実行しようかと思ったけどやっぱりやめた、などなったときに、一部のプラグインが提供するバッファのようにサクっと `q` とかで cmdwin を閉じれたら便利ですね。あるいは普通のコマンドラインのように、コマンドを入力してる途中で `<C-c>` でコマンドを実行せずに cmdwin を閉じれたら便利ですね。
マッピングしましょう。

```vim
" これらを CmdwinEnter で実行
nnoremap <buffer> q <Cmd>quit<CR>
nnoremap <buffer> <C-c> <Cmd>quit<CR>
inoremap <buffer> <C-c> <ESC><Cmd>quit<CR>
```

`<C-c>` をマッピングすることは賛否両論あると思いますが（もしかしたら賛の方はないかもしれない）、私はまあ良いだろうと思ったのでマッピングしています。`<C-c>` をマッピングしちゃうのはちょっともにょるな...という方は `<C-c>` の方のマッピングは無くしてしまってください。

### 直近の履歴のみをバッファに表示する

履歴がウインドウに一覧として出てくるのは便利ではあるのですが、履歴が多すぎるとかえって邪魔になりすぎる場合もあります。特にコマンド履歴の場合、Vim script のシンタックスが適用されるのですが、履歴が多すぎるとそのシンタックスの反映に時間がかかったりして重くなる、ということがあるようです。というわけで、ここではウインドウに表示されている履歴のうち直近のもののみを残して他をバッファから消すようにしたいと思います。バッファから行を削除するだけで、実際に履歴を消すわけではないのでそこはご安心ください。

```vim
" CmdwinEnter で以下の s:cmdwin_reduce_history() 関数を呼んでやる
function! s:cmdwin_reduce_history()
  " 直近の ('cmdwinheight' の高さ - 1) の分だけの履歴を残して、他をバッファから
  " 削除。 -1 となっているのは、最下行の空行の分です。
  execute printf('silent! 1,$-%d delete _', &cmdwinheight)

  " カーソルを最下行にもっていく
  normal! G

  " Undo のブロックをここで区切る。こうしないとコマンド編集した後、「あ、ちょっと
  " ミスったな、戻すか」と undo したときに、CmdwinEnter で消した分の履歴とかも
  " 一気に戻ってしまったりする。
  let &undolevels = &undolevels
endfunction
```

ところで余談なんですが、`let &undolevels = &undolevels` として undo を区切る方法、実は `:h undo-break` に載っている公式のやり方なんですけど、ちょっとトリッキーで面食らいますよね。

### コマンド履歴の cmdwin で <C-n> と <C-p> を使ってコマンドの補完をする

cmdwin でもコマンドラインと同じように `<Tab>` キーを使ってコマンドの補完をすることができます。これが自然に感じられるのであれば良いのですが、インサートモードでの補完なら `<C-n>` とか `<C-p>` 使った方が自然に感じられるしそうしたいなーという方もいると思います。多分私もそうです。
というわけで `<C-n>` や `<C-p>` でコマンド補完ができるようなマッピングを用意しましょう。Vim に組み込みまれている、インサートモードで使える Vim のコマンド補完の機能を使います。

```vim
" augroup は省略してます
autocmd CmdwinEnter : call s:cmdwin_completion()

function! s:cmdwin_completion()
  inoremap <buffer> <expr> <C-n>
        \ pumvisible() ? '<C-n>' : '<C-x><C-v>'
  inoremap <buffer> <expr> <C-p>
        \ pumvisible() ? '<C-p>' : '<C-x><C-v><C-p><C-p>'
endfunction
```


### ファイルを保存する/ウインドウを閉じるためのマッピングを用意する

`:w` や `:q` をするたびにいちいち cmdwin を開いて、一瞬ウインドウ全体のレイアウトが崩れてまたすぐ元に戻る、というのを繰り返すのは微妙に感じることがあると思います。これの回避策として、ファイルを保存するためのマッピングとウインドウを閉じるためのマッピングを用意してみるのはどうでしょうか。
以下の例ではファイルの保存に `<Space>w` を、ウインドウを閉じるのに `<Space>q` を割り当てていますが、みなさんお好きなキーをどうぞー。

```vim
nnoremap <Space>w <Cmd>update<CR>
nnoremap <Space>q <Cmd>quit<CR>
```

ちなみにこれ、cmdwin 使ってなくても便利だと思います。

### 最後に

今回紹介する設定もとい小ネタは以上です。cmdwin は人によって合う、合わないがあると思います。実際、cmdwin は合わなくてやめた、という話も聞いたことがあります。ただ、私はすごく便利に使わせてもらっているので、今 cmdwin を使ってる人がより便利に使えるようになったり、あるいは cmdwin に可能性を見出す・見出せないような瀬戸際にいる人が覚醒して cmdwin に目覚めるお手伝いができれば幸いです。

### おまけ

以上で紹介した設定をまとめたものを置いておきます。全部まとまっていて見通しが良くなっているものが欲しいという方はどうぞ。

```vim
" cmdwin の高さを 10 行にする
set cmdwinheight=10

" ファイルを保存する
nnoremap <Space>w <Cmd>update<CR>
" ウインドウを閉じる
nnoremap <Space>q <Cmd>quit<CR>

augroup vimrc-cmdwin
  autocmd!
  autocmd CmdwinEnter * call s:setup_cmdwin()
augroup END

function! s:setup_cmdwin()
  " ラクに cmdwin を離れる
  nnoremap <buffer> q <Cmd>quit<CR>
  nnoremap <buffer> <C-c> <Cmd>quit<CR>
  inoremap <buffer> <C-c> <ESC><Cmd>quit<CR>

  if expand('<afile>') ==# ':'
    " <C-n> や <C-p> を使ってコマンドの補完をする
    inoremap <buffer> <expr> <C-n>
          \ pumvisible() ? '<C-n>' : '<C-x><C-v>'
    inoremap <buffer> <expr> <C-p>
          \ pumvisible() ? '<C-p>' : '<C-x><C-v><C-p><C-p>'
  endif

  " 直近の履歴を残して古い履歴をバッファから消す
  execute printf('silent! 1,$-%d delete _', &cmdwinheight)
  normal! G
  let &undolevels = &undolevels  " Undo を区切る
endfunction
```
