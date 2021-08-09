---
title: "WSL2 を使った開発環境構築のまとめ"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Windows", "WSL2"]
published: false
---

# これは何か

WSL2 を使うと Windows 上で Linux な開発環境を構築することが出来るわけですが、そうなってくると IaC 的に環境を構築・再現できるようにしたくなるのが人の心というものです。
開発環境というのは秘伝のタレ的な環境になりがちですが、PC をリフレッシュするとき、環境をリセットしたいとき、に直ぐに環境を再現できるのは便利でしょう。

ということで、WSL2 で開発環境を作る方法を備忘録として残しておこうと思います。

**Docker コンテナをエクスポートして、それをそのまま WSL2 にインポートする** 、という方法で実現します。

# 参考ドキュメント・記事

いきなりですが、参考ドキュメントです。

基本的な構築方法はしばやんさんのこちらの記事を参考にするのがいいでしょう。
https://blog.shibayan.jp/entry/20210728/1627450686

Docker コンテナを WSL にインポートする方法です。今回の目的を達成するために使います。
https://docs.microsoft.com/en-us/windows/wsl/use-custom-distro


# なぜ WSL2 なのか

まず、私の開発/利用スタイルからどういう要件が必要かをまとめます。

- Docker を使いたい
- VS Code の Remote Development を使いたい
- Windows Terminal から使いたい
- 調査系のお仕事で Linux 環境を使いたい
- 主に使いたい言語・フレームワーク・ツールは以下の通り
  - 言語/フレームワーク
    - Go
    - JavaScript
    - Java
      - Spring Boot
      - Maven
      - Gradle
    - Python
  - ツール
    - Azure CLI
    - Azure Functions
    - Terraform
    - Packer
    - Ansible
    - kubectl
    - ghq / peco


これらの要件を満たす方法として、他にもいくつか方法がありますが、なぜ WSL2 を選択したのかを書いておきます。

## VM on Azure(Hyper-V/ESXi)

まず VM ですが、Azure で開発環境を作ってイメージ化しておき、使う時に VM を作る、という方法を考えました。Packer なりでゴールデンイメージを作るパターンです。

この方法では、Azure 上に VM を起動させておくことになるわけですが、当然ながら同時にコストが発生します。これがそれなりにメモリ/CPU のスペックを持つ VM になるとさらにお高めになります。こまめに電源断、サイズの変更をやってもいいのですが、使いたいときにパッと使いたい、という観点では適さないと考えました。
※実際会社のサブスクリプションになるのでそこまで制限があるわけではないのですが、他の人にも使ってもらいたい・勧めたい、場合のことを考えると1つのハードルになるのではという思いもありました

あとは、調査系の作業ゴリゴリと grep するような場面だと、野良環境にお客さんの情報を持ち出すことは出来ないのでローカルで完結する必要がありました。

他のプラットフォームとして、Hyper-V や ESXi も考えましたが、サクッと使えてデータ持ち出しの制限がない、ということを考えるとローカル環境が最適と考えました。

ただ、プラットフォームの違いはどこで環境を動かすのか、というだけなので、スクリプト化するなりで WSL2 と同じ環境を展開することは簡単にできます。なので VM on xxx のパターンを使うこと今後私の使い方によってはあり得る選択だとは思っています。

## VS Code の Dev container

VS Code の Remote Develpment では、WSL / SSH 以外に、Container が使えます。つまり、Docker Desktop for Windows をインストールしている環境で、コンテナを立てて、その中で VS Code の開発が出来るような仕組みです。

この場合、Docker のイメージを開発環境のテンプレートとして作り、開発環境が必要な時に VS Code から接続します。ビルドも使い捨ても簡単に出来て、Github Codespace でも使えるでよさそうに思いました。

ただ、私の使い方の場合、xxというアプリを作る、みたいなトランザクショナルな使い方もありますが、要件にあるようなログを grep するような場面でも使うことも多いので、揮発性の高い環境を維持し続けるというのもちょっと合わないなぁと思いました。あとはこの仕組みだと Docker の中で動くので Docker が使えないという点もマッチしませんでした。

あとは、コンテナをバカバカ作っていくとディスクの容量を割と食うのでたまにパージすることがあるのですが、そういう場合に開発環境のコンテナやイメージを消さないように気にしながら消すのは面倒だなぁというのもハマらないポイントでした。

ただ今回の方法では Docker コンテナを使うので、同時に Dev conatiner としても動かせます。


## では WSL2 はどうか

ということで、
- ローカルに Linux 環境が作れて
- ある程度永続的に使えて
- 再現性がつくれる

を考えた時に、WSL2 上に作ることが一番適している方法でした。

VM なり Dev container なりと組み合わせは出来るので、機能的な方法だけでなく運用も込みでどういう使い方をするか、という観点での選択と思ってください。

# 具体的な手順

その具体的な手順ですが、基本的な部分は参考ドキュメントを見て頂ければ事足りるのでステップバイステップで説明することは省略します。
WSL2 に取り込む部分がポイントです。

大まかに以下の通りです。

1. WSL2 のインストール
2. Docker Desktop for Windows のインストール
3. Docker コンテナのビルド
    - 開発環境をセットアップするための Dockerfile を書いて、ビルドします
4. docker export
    - ビルドしたコンテナを起動して、そのコンテナを export します
    - export したコンテナは tar.gz ファイルです
5. wsl --import
    - export した tar.gz を WSL2 に取り込みます
6. 諸々の設定・インストール
    - レジストリの変更、Windows Terminal の設定、VS Code の拡張機能のインストール等々をします

## ポイント1. Docker コンテナのビルド

ポイントとは言っても WSL2 に取り込む Docker コンテナはコンテナとして起動できれば何でもいいので、特にハマることは無いです。

参考までに私の環境の Dockerfile を貼り付けておきます。
※かなり汚い感じになってるのでもうちょっときれいにしていきたいところです

https://github.com/tsubasaxZZZ/template-devcontainer/blob/master/.devcontainer/Dockerfile

こちらを見て頂くと分かる通り、Dev conatiner(= Codespace) の環境として使えるようにもしています。
Dev container としてコンテナ環境をビルドする(かつ docker-compose を使う)場合、ユーザー周りでハマることがありましたが、他のポイントも含めこちらのドキュメントをじっくり見るといいでしょう。Dev container、いろいろ出来るので面白いです。
https://code.visualstudio.com/docs/remote/containers

## ポイント2. コンテナの export & WSL2 への取り込み

これも大したことは無いです。こちらのドキュメントをご参照ください。
https://docs.microsoft.com/en-us/windows/wsl/use-custom-distro

手順だけ抜き出すとこんな感じです。Windows 上で実行します。

```powershell
# エクスポート
cd C:\temp # tar.gz の吐き出し先に移動
docker run -t tsubasaxzzz/devcontainer ls / # コンテナを起動(作成)
docker export <コンテナID> -o dev.tar.gz # コンテナをファイルに吐き出し

# インポート
# wsl --import <ディストリビューション名> <VHDの作成先のパス> <エクスポートした tar.gz のパス>
wsl --import dev D:\wsl\dev .\dev.tar.gz # D:\wsl\dev ディレクトリが自動的に作成され、その中に VHDX ファイルが出来る
```

:::message
エクスポートするとき、コンテナのサイズによってはモリモリとメモリを消費します。WSL2(=Docker Desktop) の使うメモリ容量を調整しておいた方がいいかもしれません。
:::

ちなみに、私の今の環境です。

**Windows の VHD 置き場(取り込み先)**
これらのディレクトリの配下に `ext4.vhdx` が作成されています。
![Windows の VHD 置き場(取り込み先)](https://storage.googleapis.com/zenn-user-upload/9591e67f4735d3f3aeba4274.png)

**ディストリビューションの一覧**
![](https://storage.googleapis.com/zenn-user-upload/58e49790667ce49526788729.png)



### WSL2 にインポートした環境がいらなくなったら？

WSL2 コマンドの `--unregister` オプションを使用します。

```powershell
 $ wsl --unregister dev0807
登録を解除しています...
 $ wsl -l # dev0807 が消える(= VHDX ファイルが削除される)
Linux 用 Windows サブシステム ディストリビューション:
Ubuntu (既定)
docker-desktop-data
dev0809
docker-desktop
java0609
mikan
```

こんな感じで、WSL2 上に複数の開発環境を作って、要らなくなったらサクッと削除できます。

ちなみに、取り込んだ後に、Windows Terminal を再起動すると、このように一覧に出てくるようになります。
![](https://storage.googleapis.com/zenn-user-upload/5acf4aa4ceb4dce202c3ab48.png)


## ポイント3. WSL2 に取り込んだ後の作業

WSL2 に取り込んだ後の作業です。

- レジストリで既定のユーザー ID の変更
- Windows Terminal 起動時のディレクトリ変更
- Docker Desktop for Windows との統合
- VS Code のパス追加
- SSH の秘密鍵の設定
- ghq で参照される Git のリポジトリの参照先の設定
- VS Code の拡張インストール

特に、レジストリのユーザー ID 変更、起動時のディレクトリはいつも忘れてなんでだっけ、、とハマるので書いておきます。

### 既定のユーザー ID 変更

WSL2 の環境を root ユーザーで使う分には必要ない作業です。上で紹介した Dockerfile では tsunomur ユーザーを作成し、そのユーザー環境に諸々セットアップしているので変更しておきます。
この設定は、VS Code で Remote Development を使う場合にも関係してくるので特定のユーザーを規定にしたい場合は設定しておきます。

レジストリはこちら:
```
キー: HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss\<ユニークID>
値: DefaultUid
データ: コンテナ内での UID # 10進数/16進数を間違えないように注意
```

レジストリエディターで見るとこんな感じです。Lxss 配下に GUID のキーが生えるので中身を見てどれが追加したものかを確認します。
![](https://storage.googleapis.com/zenn-user-upload/f935ad40b9f0cba914a770f0.png)

### Windows Terminal 起動時のディレクトリ変更

Windows Terminal の起動時、既定では、`%USERPROFILE%` がディレクトリの開始ポイントになります。なので、これを Windows から見たディレクトリとして、`\\wsl$\<ディストリビューション名>\home\tsunomur` を設定します。
こんな感じ:
![](https://storage.googleapis.com/zenn-user-upload/1c3356553622f473e0c6848b.png)

### Docker Desktop for Windows との統合

WSL2 から Docker にアクセスできるように、Docker Desktop for Windows と統合します。
![](https://storage.googleapis.com/zenn-user-upload/2b5caea85c617fce7850996e.png)

### VS Code のパス追加

WSL2 の中で、`code .`みたいに VS Code を起動したいわけですが、既定だとこんなエラーが返されます。
```bash
$ code
code or code-insiders is not installed
```

:::message
WSL2 の設定で、Windows の環境変数をそのまま取り込む設定をしている場合(appendWindowsPath の設定を変更していない or true の場合)はこの手順は不要です。また、Dockerfile の FROM として、devcontainers(mcr.microsoft.com/vscode/devcontainers/base) を使っていない場合はこのエラーは表示されないはずですが同じようにパスを通す必要があります。 
:::

なので、こちらを .bashrc に書いておきます。
```bash
alias code='/mnt/c/Users/tsunomur/AppData/Local/Programs/Microsoft\ VS\ Code/bin/code'
```

# まとめ

コンテナの tar.gz を吐き出すのも手間に感じてるので、ここを何とかしたいところですが、概ね満足に使えてます。

ということで皆さまも良い WSL2 ライフを！