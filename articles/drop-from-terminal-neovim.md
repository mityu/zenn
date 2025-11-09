---
title: ":terminal から親の Vim でファイルを開く(Neovim 編)"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim"]
published: true
publication_name: vim_jp
---

:::message
この記事は [Vim 駅伝](https://vim-jp.org/ekiden/) の 2025/10/31 の記事です。
:::

# 導入

以前、Vim 駅伝の以下の記事にて、`drop` というコマンドをシェル側に実装して、ターミナルから親の Vim でファイルを開けるようにする、という話を書きました。

https://zenn.dev/vim_jp/articles/5fdad17d336c6d
https://zenn.dev/vim_jp/articles/drop-from-terminal-cmdexe

これらの記事において、私は「ここで紹介する `drop` コマンドは Vim 限定で Neovim では動きません」という趣旨のことを書いていました。
当時から Neovim への対応をして記事を出せたら良いなあと思っていたのですが、特に外部依存を使ったりすることなく簡単に実現する方法を思いつかなかったため、結局 Neovim 対応編は出せずじまいでした。
ところが最近 Neovim でも `drop` コマンドを簡単に実現できるらしいということを知ったので、今回記事にすることにしました。

# 利用する機能

Neovim の clientserver 機能を使います (`:h clientserver`)。Vim にあるそれとほぼ同じ機能です。[:terminal から親の Vim でファイルを開く(cmd.exe 編)](https://zenn.dev/vim_jp/articles/drop-from-terminal-cmdexe) で用いたものと同じですね。

コマンドラインアプリケーションとして Neovim の clientserver 機能を用いるには、コマンドラインで以下のような書式のコマンドを実行すれば良いです。

```
$ nvim -u NONE --server {サーバー名} --headless --remote-expr {Vim script の式}
```

こうすると、`{サーバー名}` で指定したインスタンスの Neovim で、`{Vim script の式}` の部分に書いた式を評価することが出来ます。
Neovim の `:terminal` 内では、`$NVIM` 環境変数に親の Neovim のサーバー名が入っているので、`:terminal` 内では、以下のような感じにしてイディオムとして使うことになるでしょう。

```
$ nvim -u NONE --server $NVIM --headless --remote-expr {Vim script の式}
```

あとは `{Vim script の式}` のところにファイルを開くための式を記述してやれば、`drop` コマンドが実装できます。

# 実装 (with Lua)

## Neovim 側

Lua の `require()` 関数でロードできる適当なファイルに、以下のような感じで `tapi_drop` 関数を定義します。
とりあえず以下では、これを `$XDG_CONFIG_HOME/nvim/lua/vimrc.lua` に記述して、ここで定義した `tapi_drop` 関数が `require('vimrc').tapi_drop(...)` として呼び出せるようになっている、として設定例を載せます。

```lua
local M = {}

--- arglist には、1番目の要素にシェルのカレントディレクトリを、2番目の要素に開く
--- ファイルのパスを入れておく
---
---@param arglist string[]
---@return string
function M.tapi_drop(_bufnr, arglist)
  local cwd = arglist[1]
  local filepath = arglist[2]
  if vim.fn.isabsolutepath(filepath) == 0 then
    -- 絶対パスでない時は絶対パスに変換する
    filepath = vim.fs.joinpath(cwd, filepath)
  end

  -- ファイルがすでに開かれていればそのウインドウに移動する
  local opencmd = 'drop'
  if vim.fn.bufwinnr(vim.fn.bufnr(filepath)) == -1 then
    -- ファイルがすでに開かれている場合でなければウインドウを分割して開く
    opencmd = 'split'
  end
  vim.cmd(opencmd .. ' ' .. vim.fn.fnameescape(filepath))
  return ''
end

return M
```

これは、[:terminal から親の Vim でファイルを開く(bash/zsh編)](https://zenn.dev/vim_jp/articles/5fdad17d336c6d) で書いた `Tapi_drop` 関数を Lua に移植したものです。
が、一点だけ変更していて、関数の末尾で空文字列を返すようにしています。これは、Neovim の clientserver 機能が `--remote-expr` 引数で与えた式の評価結果をターミナルに出力するからで、今回は特にターミナルに出力させたいものがないので明示的に空文字列を返しています。


## シェル側

Neovim の `:terminal` 内では、`$NVIM` 環境変数が定義されているので、その環境変数の有無で `drop` コマンドを定義するかどうかを分岐します。
また、Neovim では `v:lua` 変数を用いて Vim script 側から Lua 側の関数を呼ぶことが可能なので、`tapi_drop` 関数はこれを用いて `v:lua.require('vimrc').tapi_drop(...)` のようにして呼び出します。

### bash/zsh

bashrc/zshrc などに以下の内容を足します。

```bash
if [ -n "$NVIM" ]; then
  function drop() {
    nvim -u NONE --server $NVIM --headless --remote-expr "v:lua.require('vimrc').tapi_drop(0, ['$(pwd)', '$1'])"
  }
fi
```

### fish

`config.fish` に以下の内容を足します。

```fish
if string length -q -- $NVIM
  function drop
    nvim -u NONE --server $NVIM --headless --remote-expr "v:lua.require('vimrc').tapi_drop(0, ['$(pwd)', '$argv[1]'])"
  end
end
```

### cmd.exe

以下のような内容で `drop.bat` を作成し、パスの通っているところに置きます。

```
@echo off
REM :terminal 内じゃなければスキップ
if "%NVIM%" == "" goto :EOF
nvim -u NONE --server %NVIM% --headless --remote-expr "v:lua.require('vimrc').tapi_drop(0, ['%cd%', '%1'])"
```

::::details Vim 版との共存
おそらく `drop.bat` の内容をこうすれば良いです。(が、色々あって私は Windows 環境が飛んでしまったので動作確認ができていません...。)
```
@echo off
REM Vim :terminal 内じゃなければ Neovim の :terminal 内かどうかチェックしにいく
if "%VIM_TERMINAL%" == "" goto checknvim
vim --servername %VIM_SERVERNAME% --remote-send "<Cmd>call Tapi_drop(0, ['%cd%', '%1'])<CR>"
goto :EOF

checknvim:
REM :terminal 内じゃなければスキップ
if "%NVIM%" == "" goto :EOF
nvim -u NONE --server %NVIM% --headless --remote-expr "v:lua.require('vimrc').tapi_drop(0, ['%cd%', '%1'])"
```
::::

# 実装 (with Vim script)


## Neovim 側

init.vim 等で、以下のような `Tapi_drop` 関数を定義します。

```vim
function! Tapi_drop(bufnr, arglist) abort
  let cwd = a:arglist[0]
  let filepath = a:arglist[1]
  if isabsolutepath(filepath)
    " 絶対パスでない時は絶対パスに変換する
    let filepath = fnamemodify(cwd, ':p') . filepath
  endif

  " ファイルがすでに開かれていればそのウインドウに移動する
  let opencmd = 'drop'
  if bufwinnr(bufnr(filepath)) == -1
    " ファイルがすでに開かれている場合でなければウインドウを分割して開く
    let opencmd = 'split'
  endif
  execute opencmd fnameescape(filepath)
  return ''
endfunction
```

上の「実装 (with Lua)」で書いた `tapi_drop` をそのまま Vim script で実装したものです。

## シェル側

上の「実装 (with Lua)」にて `--remote-expr` の引数が `"v:lua.require('vimrc').tapi_drop(...)"` となっていたのを、`"Tapi_drop(...)"` となるように変更すれば良いです。

# おわりに

というわけで Neovim 側にも `drop` コマンドを導入することができました。いやー、相変わらず便利ですね。
