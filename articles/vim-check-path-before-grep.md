---
title: "Vim/Neovim で grep 前にディレクトリの確認を入れると便利"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "neovim"]
published: true
published_at: 2025-10-06 00:00
publication_name: "vim_jp"
---

この記事は [Vim 駅伝](https://vim-jp.org/ekiden/) の 2025/10/06 の記事です。
前回の記事は atusy さんの「[countを使ってeasymotionとかでfモーションを拡張した時に組込みのfモーションも残す](https://blog.atusy.net/2025/10/03/switch-to-native-mapping-with-count/)」 でした。

---

# はじめに

例えば、[kyoh86/vim-ripgrep](https://github.com/kyoh86/vim-ripgrep) など、世の中には Vim から grep を行うためのプラグインがいくつかあります。
シェルのコマンドラインで grep するときは、grep したいディレクトリに移動してから grep するので発生しないのですが、Vim からこういったプラグインで grep をするときは自分が思っていたのとは異なるディレクトリで grep してしまった、ということがままあります。これを防ぐのに、grep 前に grep するディレクトリの確認ダイアログを出すようにすると便利です。

# 設定例

上で例に挙げた [kyoh86/vim-ripgrep](https://github.com/kyoh86/vim-ripgrep) に組み込む例です。
上から順に `s:get_default_grep_cwd` 関数、`s:ripgrep` 関数、`:RipGrep` コマンドが定義されていて、それぞれ

- `s:get_default_grep_cwd` 関数
  - Grep するときのデフォルトのディレクトリを取得する関数です
- `s:ripgrep` 関数
  - Grep するディレクトリの確認をしたあと、引数で与えられた検索クエリで grep する関数です
    - `:RipGrep` コマンドから呼び出されます
    - 引数で検索クエリが与えられなかったとは、ディレクトリ確認のダイアログを出したあと、検索クエリ入力プロンプトを出します
- `:RipGrep` コマンド
  - `s:ripgrep` 関数を呼び出して grep するコマンドです
    - コマンド引数で検索クエリを指定します
    - 指定しなかったときは、あとで検索クエリ入力プロンプトがでます。

という感じになっています。

```vim
" Grep する時のディレクトリのデフォルト候補を取得する関数。
" この例が返すディレクトリは :grep と同じになっているハズ。
function s:get_default_grep_cwd() abort
  return getcwd(winnr())
endfunction

function s:ripgrep(args) abort
  let cwd = s:get_default_grep_cwd()
  let cwd = input('Grep here? ', cwd, 'dir')->expand()
  if !isdirectory(cwd)
    echoerr $'Directory does not exist: {cwd}'
    return
  endif

  let args = a:args
  if args ==# ''
    let args = input('Search pattern: ')
  endif

  let prefix = fnamemodify(fnamemodify(cwd, ':.'), ':p')
  let command = $'rg --json -i {args}'

  " バックスラッシュとかが絡むと、自分の想定していない検索クエリが飛んでいる場
  " 合があるので、投げる grep コマンドを表示しておくようにすると便利です。
  echomsg $'Executing: {command}'
  call ripgrep#call(command, cwd, prefix)
endfunction

command! -bar -nargs=? RipGrep call s:ripgrep(<q-args>)
```

# おわりに

grep ごとにディレクトリの確認をはさむと、grep コマンドを叩くときの冗長さが増えることにもなり欠点もあります。が、私はその欠点よりも必ず自分の思ったディレクトリで grep コマンドを叩ける利点の方が大きいと感じられるのでこれを導入しています。
また、たまに発生する「今の作業ディレクトリとは全然違うところで grep した〜い」というときに、痒いところに手が届く感じで非常に便利です。

# おまけ1

grep したいディレクトリとは違うディレクトリが出てきて、パスの編集をしたいとなったときの tips を少しだけ。

Vim にはコマンドラインにおいて特殊な意味を持つ文字があり、主にカレントバッファに関連するパスを簡単に入力することができます。
ざっとかいつまんで説明すると、Vim のコマンドラインでは `%` がカレントバッファのファイルパスを意味し、`<Tab>` キーを押すことで具体的なパスに展開することができます。
また、その `%` に続けて `:h` や `:p` を繋げて書いてから `<Tab>` キーを押すことで、カレントバッファのファイルパスの親ディレクトリや、あるいはそのパスを絶対パスに変換したものとして展開することができます。

`%` がカレントバッファのファイルパスを意味する、という話は `:h cmdline-special` に記述があります。`%` 以外にも使える文字が載っているので気になる方はどうぞ。また、`:h` や `:p` などは "filename-modifiers" と呼ばれるもので、`:h filename-modifiers` に記述があります。
この記法は、例えば `:e` で別ファイルを開く際など、今回の grep の用途以外でも絶大な効果を発揮する場面があるので、覚えておいて損はないと思います。

とりあえずよく使うパターンについて具体例を置いておきます。今回の grep の用途ではこれだけを覚えていれば大半のケースで事足りると思うので、とりあえずこれらから覚えていくのが良いでしょう。 

- カレントバッファで開いているファイルの親ディレクトリを入力する
  `%:h<Tab>`
- カレントバッファで開いているファイルの2つ親のディレクトリを入力する
  `%:h:h<Tab>`
- カレントバッファで開いているファイルの3つ親のディレクトリを入力する
  `%:h:h:h<Tab>`
- and so on...

# おまけ2...にしたかったもの

`:vimgrep` とか `:grep` で同じことをするにはこうすれば良いよ！というのを書きたかったんですが、うまいこと `:vimgrep` とかにフックしてカレントバッファの作業ディレクトリを変更し、grep が終わったら戻して、というのを綺麗に（変な副作用なしに）することができなかったので断念しました。
誰かできる方がいらっしゃいましたら教えてください。というかせっかくなので、ぜひそのネタで Vim 駅伝に投稿してください。
