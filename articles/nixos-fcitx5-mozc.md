---
title: "NixOS に入れた fcitx5 で、mozc を自動で有効にしておく"
emoji: "✒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nix", "NixOS", "fcitx5", "Mozc"]
published: true
published_at: 2025-09-23 07:30
---

昔ハマったので備忘録がてらに。
なお、以下では fcitx5 は home-manager でユーザーランドにインストールしたものとします。

# 問題

とりあえず home-manager を使って fcitx5 を mozc パッケージ付きでインストールするには、以下のような記述をすれば良いです。

```nix
i18n.inputMethod = {
  enable = true;
  type = "fcitx5";
  fcitx5.addons = [ pkgs.fcitx5-mozc ];
};
```

これで一応ちゃんと fcitx5 を mozc 付きでインストールすることはできるのですが、実際に mozc を fcitx5 から扱おうとすると、下の画像のように、GUI による fcitx5 の設定ツール `fcitx5-configtool` を使って手動で mozc を入力ソースに追加にしてやらないと、mozc が有効にならず使えないという状況になっていました。

![](/images/nixos-fcitx5-mozc-fcitx5-configtool.png)
*fcitx5 の設定ツール `fcitx5-configtool`*

私は使う IM の構成は決め打ちなので、せっかく Nix を使うのであればこの mozc を有効にする設定もどうにか Nix 側で記述できたらうれしいです。


# 解決策

home-manager の `i18n.inputMethod.fcitx5.settings.inputMethod` オプションを設定します。
https://mynixos.com/home-manager/option/i18n.inputMethod.fcitx5.settings.inputMethod

上のページの説明を見ればわかるのですが、このオプションは、記述された内容で `~/.config/fcitx5/profile` の内容を置き換えます。
この `~/.config/fcitx5/profile` というファイルは、fcitx5 が使用する入力ソースの情報をまとめたものなので、これを適当な内容に設定しておくことで、最初から mozc が有効になった状態の fcitx5 を手に入れることができます。
言い換えると、このオプションで、上の `fcitx5-configtool` のスクリーンショットの "Input Method" タブの内容を Nix 側で設定できるということですね。

実際にこのオプションに設定する値に関しては、一度 `fcitx5-configtool` で fcitx5 の設定を行ったあと、`~/.config/fcitx5/profile` を確認してその内容を Nix 式に書き起こす、という形で用意するのが一番手っ取り早いと思います。

参考までに 2025/09/23 現在の私の設定を置いておきます。fcitx5 の構成はこんな感じです。

- 入力ソース1: 英数モード（US レイアウト）
    - （もしかしたら IM がオフのモードと言った方が正確かもしれません）
- 入力ソース2: mozc

```nix:home.nix
i18n.inputMethod.fcitx5.settings.inputMethod = {
  GroupOrder = {
    "0" = "Default";
  };
  "Groups/0" = {
    Name = "Default";
    "Default Layout" = "us";
    DefaultIM = "mozc";
  };
  "Groups/0/Items/0" = {
    Name = "keyboard-us";
    Layout = "";
  };
  "Groups/0/Items/1" = {
    Name = "mozc";
    Layout = "";
  };
};
```

この設定で、以下のような `~/.config/fcitx5/profile` が出力されます。

```:~/.config/fcitx5/profile
[GroupOrder]
0=Default

[Groups/0]
Default Layout=us
DefaultIM=mozc
Name=Default

[Groups/0/Items/0]
Layout=
Name=keyboard-us

[Groups/0/Items/1]
Layout=
Name=mozc
```

# 注意

この `i18n.(略).inputMethod` オプションは便利なのですが、少し注意点もあります。
このオプションは `~/.config/fcitx5/profile` ファイルの中身を**完全に置き換える**ため、

 - `~/.config/fcitx5/profile` の内容を全て Nix 式に変換しないといけない
 - `fcitx5-configtool` を使って変更した IM の設定が、home-manager の設定の再ビルド後に消える
    - なので、その変更を永続化したければ home-manager の設定ファイルにバックポートしないといけない

というような話があります。ご注意をば。
Nix あるあるの話だと言われるとそうかもしれないですね。

# 余談1

nixpkgs 側 (flake の nixosConfigurations 側？) にも全く同じ fcitx5 オプションがあるので、home-manager 側ではなく nixpkgs 側でこの fcitx5 の設定を書いてインストールしてもうまくいくかもしれません（少なくともビルドは通りそうな雰囲気があります）。が、私は試していないので、実際に nixpkgs 側に設定を記述してうまいこといくのかはよくわかりません。
とりあえず nixpkgs 側にも同じオプションがあるっぽいよ〜という情報だけ残しておきます。

# 余談2

今回の `i18n.(略).inputMethod` に似たオプションとして、`i18n.inputMethod.fcitx5.settings.globalOptions` というオプションもあります。
https://mynixos.com/home-manager/option/i18n.inputMethod.fcitx5.settings.globalOptions

こちらは `~/.config/fcitx5/config` ファイルの内容を設定するオプションで、fcitx5 に関するキーコンフィグなどを記述できます。記事冒頭の `fcitx5-configtool` のスクリーンショットでは、"Global Options" となっているタブの内容を設定できるオプションです。合わせて確認すると幸せになれるかもしれません。記述の仕方も `i18n.(略).inputMethod` オプションとそっくりなので、上に書いた `fcitx5-configtool` で設定してから Nix に起こす、というやり方が通用すると思います。
ただ、こちらも `i18n.(略).inputMethod` オプションと同じで `~/.config/fcitx5/config` ファイルを**完全に置き換える**オプションなので、前述した注意が同様に当てはまることには留意してください。

# 最後に

誰かハマった方の助けになれば。
