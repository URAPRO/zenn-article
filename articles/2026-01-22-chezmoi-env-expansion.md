---
title: "chezmoi環境変数展開でClaude Code起動を楽にした話"
emoji: "🔐"
type: "tech"
topics: ["chezmoi", "dotfiles", "1password", "claude"]
published: false
---

# 🔐 chezmoi環境変数展開でClaude Code起動を楽にした話

## はじめに

以前、[chezmoiでdotfiles管理を始めた記事](https://urapro.dev/posts/chezmoi/)を書きました。1Password連携で秘密情報を安全に管理できるようになって、満足していたんですよね。

でも、運用していくうちに「ちょっと面倒だな...」と感じる部分が出てきました。

今回はその改善の話です。

## op run方式の課題

[以前の記事](https://zenn.dev/genda_jp/articles/e8e2a2df13643e)で、1Password CLIの`op run`を使って環境変数を注入する方法を紹介しました。設定ファイルに秘密情報をベタ書きしなくて済むので、セキュリティ的には良い方法だなと思っています。

ただ、運用していく中で一つ問題があり。これだと、起動のたびに1Passwordのロック解除が必要になります。

Claude Codeって、セッションを意図的にクリアすることがあるんですよね。コンテキストがいっぱいになったり、話題を切り替えたいときとか。あと、僕は複数のAPIプロバイダを切り替えて使ったりもするので、起動頻度がそこそこ高いんです。

そのたびにTouch IDが〜とか、1Passwordの入力が〜とか、地味に面倒です。

## 解決策: chezmoi applyで展開する

考えたのは、「認証のタイミングを変える」というアプローチです。

- **Before**: ツール起動時に毎回認証 → 頻度高い
- **After**: `chezmoi apply`時に認証 → 1日1回程度

chezmoiにはテンプレート機能があって、ファイル展開時に1Passwordから値を取得できます。これを使えば、**Gitにはテンプレートファイルだけをコミット**して、**ローカルには展開済みの値**を置いておける。

シェル起動時には、もう展開された環境変数ファイルを読み込むだけなので、認証は不要というわけです。

## 実装方法

### 1. テンプレートファイルを作成

`~/.local/share/chezmoi/dot_zshrc.d/02-credentials.zsh.tmpl` を作成します。

```zsh
# Credentials (1Password から取得)
# chezmoi apply 時に1Password認証が必要、シェル起動時は不要

# Slack MCP Token (個人アカウント: my.1password.com)
export SLACK_USER_TOKEN="{{ onepasswordRead "op://Private/Slack MCP User Token/user_token" "my.1password.com" }}"
```

ポイントは `.tmpl` 拡張子ですね。chezmoiはこの拡張子を見て、テンプレート処理が必要なファイルだと判断します。

### 2. 1Passwordにアイテムを登録

1Passwordに以下の構造でアイテムを作成しておきます。

- **Vault**: Private
- **アイテム名**: Slack MCP User Token
- **フィールド名**: user_token
- **値**: 実際のトークン

### 3. chezmoi applyを実行

```bash
chezmoi apply
```

初回はTouch ID認証を求められます。認証が通ると、`~/.zshrc.d/02-credentials.zsh`（`.tmpl`なし）が生成されて、中身には実際のトークンが展開されています。

```zsh
# 展開後の中身（実際の値が入っている）
export SLACK_USER_TOKEN="xoxp-xxxxx-xxxxx-xxxxx"
```

### 4. .zshrcで読み込み

僕は`.zshrc`で`.zshrc.d/`配下のファイルを順番に読み込む構成にしているので、シェル起動時に自動で環境変数がセットされます。

```zsh
# ~/.zshrc
for config in ~/.zshrc.d/*.zsh; do
  source "$config"
done
```

これで、シェルを開くたびに認証なしで環境変数が使えるようになりました。

## Gitには何がコミットされるか

ここが重要なポイントですね。

Gitリポジトリには `.tmpl` ファイルだけがコミットされます。

```
dot_zshrc.d/02-credentials.zsh.tmpl  ← これがGitにある
```

中身は `{{ onepasswordRead ... }}` というテンプレート記法だけなので、実際のトークンは含まれていません。

一方、ローカルの `~/.zshrc.d/02-credentials.zsh` には展開された値が入っていますが、このファイルはchezmoiの管理対象（ターゲット）であってソースではないので、Gitには入らない。

```
~/.zshrc.d/02-credentials.zsh  ← ローカルにはある（Gitにはない）
```

## セキュリティ的にどうなの？

正直なところ、「ローカルファイルに秘密情報が平文で残る」というのは気になる人もいると思います。

僕の考えとしては：

1. **Gitには残らない** → リモートリポジトリに秘密情報が漏れる心配はない
2. **ローカルマシンの保護は別レイヤー** → FileVaultやログインパスワードでマシン自体を保護
3. **認証頻度と利便性のトレードオフ** → 毎回認証するよりは、1日1回の方が現実的

完璧なセキュリティと完璧な利便性は両立しないので、どこでバランスを取るかという話ですね。個人の開発環境としては、このくらいが落としどころかなと思っています。

> **※ ただし、共有PCや持ち出しPCの場合は別途検討が必要です**

## まとめ

- `op run`方式は起動のたびに認証が必要で面倒だった
- chezmoiの`onepasswordRead`を使えば、apply時に展開できる
- Gitにはテンプレートだけ、ローカルには展開済みの値
- 認証は`chezmoi apply`の時だけ（1日1回程度）

Claude Codeをガンガン使う人には、この方式おすすめですよ。

参考になれば幸いです。
