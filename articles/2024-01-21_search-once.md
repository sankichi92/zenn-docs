---
title: "複数の Web サイトを CLI からまとめて検索する"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "cli"]
published: true
---

この記事では、`search-once` という Rust 製の CLI アプリケーションを紹介します。

https://github.com/sankichi92/search-once

## 背景

職場で、チケット管理ツールや社内 Wiki、ファイル共有システムをまとめて検索したいという声がありました。
実際、次のような場合に、こうしたツールをひとつずつ検索してくことはよくあります。

- hogehoge に関する議論をどこかで見た記憶があるけど、どこだったかわからない
- fugafuga というエラーが出たけど、同じエラーに遭遇した他の人はいないだろうか

同じクエリをサービスごとに打ち込んで検索するのは面倒なので、クエリを1回打ち込むだけでまとめて検索できると便利そうです。

## つくったもの

これを実現するのが `search-once` です[^1]。

[^1]: UI を考えるのが面倒だったのと、Crates.io で Rust パッケージをリリースしてみたかったのとで、Rust 製 CLI アプリになりました。単に社内向けであれば、ペライチ Web アプリの方が非エンジニアにも使えて便利そうです。

### インストール

Rust のパッケージマネージャ Cargo を使ってインストールできます[^2]。

[^2]: 小さなアプリで簡単にビルドできるため、バイナリ配布はしていません。

```console
$ cargo install search-once
```

### 使い方

`seach-once <QUERY>` で、設定したサイトの検索結果画面をまとめて開きます[^3]。

[^3]: [wslu](https://wslutiliti.es/wslu/install.html) の `wslview` コマンドがインストールされていれば、WSL からも利用できます。

```console
$ search-once hoge
Config: /Users/sankichi92/Library/Application Support/rs.search-once/default-config.yml
github-rust:    https://github.com/search?q=language%3ARust+hoge&type=repositories
crates-io:      https://crates.io/search?q=hoge
```

実行すると、標準出力に設定ファイルのパスが表示されるので、当該ファイルを編集してください。
上記は macOS の例で、デフォルトの設定ファイルのパスは OS によって異なります。

設定ファイルのフォーマットは、以下のような YAML です。

```yaml
---
sites:
  - name: github-rust
    url: https://github.com/search?q=language%3ARust+%s&type=repositories
  - name: crates-io
    url: https://crates.io/search?q=%s
```

`url` の値の `%s` 部分がクエリ文字列に置き換えられます。
これは、[Chrome のサイト内検索ショートカット](https://support.google.com/chrome/answer/95426)のフォーマットと同じです。

`--config` オプションで設定ファイルのパスの指定もできます。
その他のオプションは `search-once --help` の出力を参照してください。

```console
$ search-once --help
A tool to search multiple websites at once

Usage: search-once [OPTIONS] <QUERY>

Arguments:
  <QUERY>  

Options:
  -c, --config <FILE>  
  -h, --help           Print help
  -V, --version        Print version
```

## 実装

実装は `main.rs` のバイナリクレートのみで、全部で100行にも満たないシンプルなものです。

https://github.com/sankichi92/search-once/blob/main/src/main.rs

---

[Rust の CLI Book](https://rust-cli.github.io/book/index.html) を参照しながら、はじめて Rust で CLI アプリをつくりました。
Ruby 等のスクリプト言語に比べ、ユーザ入力や I/O でエラーの起こりうる箇所に自然と意識が向き、やはり安心してコードが書けます。
また、[clap](https://docs.rs/clap/) クレートによる CLI 設計や、[anyhow](https://docs.rs/anyhow) クレートによるエラーハンドリングは気が利いていて、CLI アプリの開発体験は非常によかったです。
