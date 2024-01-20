---
title: "Vim/Neovim でカーソル下のハイライトの情報を表示する"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim"]
published: true
published_at: 2024-01-26 00:00
publication_name: "vim_jp"
---

この記事は [Vim 駅伝](https://vim-jp.org/ekiden/) 2024/1/26 の記事です。

# 成果物

カーソル下のハイライトの情報を以下の画像の様な感じでイイカンジに表示してくれる関数を書きました。

![](/images/vim-show-hlgroup-demo.png)

# 実装

```vim
function ShowHighlightGroup()
  let hlgroup = synIDattr(synID(line('.'), col('.'), 1), 'name')
  let groupChain = []

  while hlgroup !=# ''
    call add(groupChain, hlgroup)
    let hlgroup = matchstr(trim(execute('highlight ' . hlgroup)), '\<links\s\+to\>\s\+\zs\w\+$')
  endwhile

  if empty(groupChain)
    echo 'No highlight groups'
    return
  endif

  for group in groupChain
    execute 'highlight' group
  endfor
endfunction
```

あとはこの関数をコマンドラインから直接呼び出すなり、この関数を呼び出すコマンドを定義するなりご自由にどうぞ。私は `:Hlgroup` という名前のコマンドにして使っています。

```vim
command! -bar Hlgroup call ShowHighlightGroup()
```

:::details おまけ: Vim9 script版
```vim
def ShowHighlightGroup()
  var hlgroup = synIDattr(synID(line('.'), col('.'), 1), 'name')
  var groupChain: list<string> = []

  while hlgroup !=# ''
    groupChain->add(hlgroup)
    hlgroup = execute($'highlight {hlgroup}')->trim()->matchstr('\<links\s\+to\>\s\+\zs\w\+$')
  endwhile

  if empty(groupChain)
    echo 'No highlight groups'
    return
  endif

  for group in groupChain
    execute 'highlight' group
  endfor
enddef
```

(ちなみに私は vimrc を Vim9 script で書いているので、実はこちらが本家だったりします。)
:::

# 作った動機

`:h synID()` を見ると、カーソル下のハイライトグループの名前を表示するためのコマンドが例として紹介されています。

> 例(カーソルの下の構文アイテムの名前を表示する):
>    :echo synIDattr(synID(line("."), col("."), 1), "name")

ただ、これで得られるのはハイライトグループの名前だけで、実際にハイライトに使われているカラーコードを知りたい場合はさらに `:highlight XXX` のようなコマンドを実行したりする必要があり、少し手間です。
また、そのハイライトの色が `:highlight link XXX YYY` のような感じで、他のハイライトグループにリンクする形で定義されているときなどは、追加で `YYY` の色を参照しにいく必要があります。さらに `YYY` も別のハイライトグループ `ZZZ` にリンクされている場合はさらに `ZZZ` の色情報を見に行って...という感じで、色が別のハイライトグループにリンクする形で定義されている間、繰り返し `:highlight` コマンドを叩く必要があります。
これが面倒なので、1. カーソル下のハイライトグループの取得 2. 実際に使われているカラーコードがわかるまでハイライトグループのリンクをたどる 3. 取得したハイライトグループの色情報を表示する、の一連の作業を一気に行う関数を作りました。

# おわりに

ちょっと痒い所に手が届くような感じで地味に便利です。この間唐突に本体に `Added`・`Removed`・`Changed` というハイライトグループが追加されたとき、カラースキームをいじるのに役立ちました。
