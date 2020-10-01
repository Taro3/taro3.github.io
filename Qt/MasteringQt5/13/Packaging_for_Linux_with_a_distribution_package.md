# ディストリビューションパッケージを使ったLinuxのパッケージ化

Linux ディストリビューション用のアプリケーションをパッケージ化するのは、でこぼこ道です。それぞれのディストリビューションは独自のパッケージングフォーマット(.deb、.rpmなど)を持つことができるので、最初に答えるべき質問は、どのディストリビューションをターゲットにしたいかということです。すべての主要なパッケージングフォーマットを網羅するには、いくつかの章が必要です。単一のディストリビューションを詳細に説明することさえ不公平かもしれません (あなたは RHEL のためにパッケージ化したいと思っていましたか？ 残念ながら、私たちは Arch Linux だけをカバーしていました！)。結局のところ、Qt アプリケーション開発者の視点から見ると、あなたが望んでいるのはユーザに製品を出荷することであり、 (まだ) 公式の Debian リポジトリメンテナになることを目指しているわけではありません。

このようなことを念頭に置いて、私たちは各ディストリビューションのためにアプリケーションをパッケージ化するツールに焦点を当てることにしました。そうです、あなたは Debian や Red Hat の内部を学ぶ必要はありません!それでも、パッケージ化システムに共通する原理については、 過度な詳細なしに説明します。

私たちの目的のために、我々はUbuntuマシン上で.deb形式を使用してパッケージングを行うことができる方法を示しますが、あなたが見ることができるように、それは簡単に.rpmを生成するために更新することができます。

これから使うツールは、fpm (**eFfing Package Management**) という名前のものです。

***

## Info

fpm ツールは https://github.com/jordansissel/fpm から入手できます。

***

fpm ツールは、ディストリビューション固有の詳細の世話をして最終的なパッケージを生成するという、まさに私たちが必要としていることを行うことを目的とした Ruby アプリケーションです。まず、時間をかけて fpm をマシンにインストールし、動作していることを確認してください。

一言で言えば、Linux パッケージとは、デプロイしたいすべてのファイルを多くのメタデータと一緒に含んだファイル形式のことです。内容の説明、変更履歴、ライセンスファイル、依存関係のリスト、チェックサム、インストール前後のトリガー、その他多くのものを含むことができます。

***

## Info2

Debian バイナリを手作業でパッケージ化する方法を学びたい場合は、 <http://tldp.org/HOWTO/html_single/Debian-Binary-Package-Building-HOWTO/> にアクセスしてください。

***

今回のケースでは、fpm に仕事をさせるためにプロジェクトの準備をしなければなりません。デプロイしたいファイルはターゲットのファイルシステムと一致していなければなりません。デプロイは以下のようになります。

* gallery-desktop: このバイナリは /usr/bin にデプロイしてください。
* libgallery-core.so: これは/usr/libに配置する必要があります。

これを実現するために、出力を dist/desktop-linux で以下のように整理します。

* build ディレクトリには、コンパイルされたプロジェクトが格納されます (これは、私たちのリリースシャドウビルドです)。
* root ディレクトリにはパッケージ化予定のファイル、つまり適切な階層 (usr/bin と usr/lib) にあるバイナリファイルとライブラリファイルが格納されます。

ルートディレクトリを生成するために、Qtと.proファイルの力に頼ります。Qtプロジェクトをコンパイルするとき、ターゲットファイルはすでに追跡されています。あとは、gallery-coreとgallery-desktop用に追加のインストールターゲットを追加するだけです。

gallery-core/gallery-core.proに以下のスコープを追加します。

```QMake
linux {
    target.path = $$_PRO_FILE_PWD_/../dist/desktop-linux/root/usr/lib/
    INSTALLS += target
}
```

ここでは、DISTFILES（.soファイル）を目的のルートツリーにデプロイする新しいtarget.pathを定義しています。現在の .pro ファイルが保存されているディレクトリを指す $$_PRO_FILE_PWD_ の使用に注意してください。

gallery-desktop/gallery-desktop.proでもほぼ同じ手順で実行されます。

```QMake
linux {
    target.path = $$_PRO_FILE_PWD_/../dist/desktop-linux/root/usr/bin/
    INSTALLS += target
}
```

これらの行で、make install を呼び出すと、ファイルは dist/desktop-linux/root/.... にデプロイされます。

プロジェクトの設定が完了したので、パッケージングスクリプトに切り替えます。ここでは、スクリプトを2回に分けて説明します。

* プロジェクトのコンパイルとrootの準備。
* fpmによる.debパッケージの生成。

まず、環境変数が正しく設定されていることを確認します。

![image](img/2.png)

scripts/package-linux-deb.shを以下の内容で作成します。

```shell
#!/bin/bash

DIST_DIR=dist/desktop-linux
BUILD_DIR=build
ROOT_DIR=root

BIN_DIR=$ROOT_DIR/usr/bin
LIB_DIR=$ROOT_DIR/usr/lib

mkdir -p $DIST_DIR && cd $DIST_DIR
mkdir -p $BIN_DIR $LIB_DIR $BUILD_DIR

pushd $BUILD_DIR
$QTDIR/bin/qmake \
    -spec linux-g++ \
    "CONFIG += release" \
    ../../../ch13-gallery-packaging.pro

make qmake_all
pushd gallery-core && make && make install ; popd
pushd gallery-desktop && make && make install ; popd
popd
```

これを分解してみましょう。

1. メインパスの変数を設定します。出力ディレクトリはdist_dirです。全てのファイルは dist/desktop-linux/build フォルダに生成されます。
2. 全てのディレクトリを作成し、dist/desktop-linux/build を起動します。
3. Linuxプラットフォームのリリースモードでqmakeを実行して、親プロジェクトのMakefileを生成します。
4. make qmake_all コマンドを実行して、サブプロジェクトの Makefile を生成します。
5. make コマンドを実行して、必要な各サブプロジェクトを構築します。
6. make install コマンドを使用して、バイナリとライブラリを dist/desktop-linux/root ディレクトリにデプロイします。

scripts/package-linux-deb.sh を実行すると、dist/desktop-linux の最終的なファイルツリーは以下のようになります。

* build/
  * gallery-core/*.o
  * gallery-desktop/*.p
  * Makefile
* root/
  * usr/bin/gallery-desktop
  * usr/lib/libgallery-core.so

これで fpm が動くようになりました。scripts/package-linux-deb.sh の最後の部分には次のように書かれています。

```shell
fpm --input-type dir \
    --output-type deb \
    --force \
    --name gallery-desktop \
    --version 1.0.0 \
    --vendor "Mastering Qt 5" \
    --description "A Qt gallery application to organize and manage your
pictures in albums" \
    --depends qt5-default \
    --depends libsqlite3-dev \
    --chdir $ROOT_DIR \
    --package gallery-desktop_VERSION_ARCH.deb
```

ほとんどの論拠は十分に明示されています。最も重要なものに焦点を当てる。

* --input-type: この引数は、fpm がどのような形式で動作するかを指定します。deb, rpm, gem, dir などを受け取り、別の形式にリパッケージすることができます。ここでは dir オプションを使って、ディレクトリツリーを入力ソースとして使うように fpm に指示しています。
* --output-type: この引数は、希望する出力タイプを参照します。どのくらいのプラットフォームに対応しているかについては、公式ドキュメントを見てください。
* --name: これはパッケージに与えられた名前です（アンインストールしたい場合はapt-get remove gallery-desktopと書きます）。
* --depends: この引数は、プロジェクトのライブラリパッケージの依存関係を参照します。必要な数だけ依存関係を追加することができます。この例では、qt5 -defaultとsqlite3-devだけに依存しています。このオプションは非常に重要なので、アプリケーションがターゲットプラットフォーム上で実行できることを確認してください。依存関係のバージョンは --depends library >= 1.2.3 で指定できます。
* --chdir: この引数は、fpm を実行するベースディレクトリを指定します。ここでは dist/desktop-linux/root に設定して、ファイルツリーを読み込む準備ができました！
* --package: この引数は最終的なパッケージの名前です。VERSION と ARCH は、あなたのシステムに基づいて自動的に埋められるプレースホルダです。

残りのオプションは純粋に参考になるものです。変更履歴やライセンスファイルなどを指定することができます。output-typedeb を rpm に変更するだけで、パッケージフォーマットが適切に更新されます。fpm ツールは特定のパッケージフォーマットオプションも提供しており、生成されるものを細かく制御することができます。

scripts/package-linux-deb.sh を実行すると、新しい dist/desktop-linux/gallery-desktop_1.0.0_amd64.deb ファイルが生成されるはずです。コマンドでインストールしてみてください。

```shell
    sudo dpkg -i dist/desktop-linux/gallery-desktop_1.0.0_amd64.deb
    sudo apt-get install -f
```

最初のコマンドはパッケージをシステムにデプロイします。これで、/usr/bin/gallery-desktop と /usr/lib/libgallery-core.so というファイルができたはずです。

しかし、dpkg コマンドを使ってパッケージをインストールしたため、依存関係は自動的にインストールされません。これは、パッケージが Debian リポジトリから提供されている場合に行われます (このため、apt-get install gallery-desktop でパッケージをインストールします)。不足している依存関係は「マーク」されたままで、apt-get install -f がインストールを行います。

gallery-desktopというコマンドで、システムのどこからでもgallery-desktopを起動できるようになりました。2016年にこの章を書いたときは、「新鮮な」Ubuntuで実行すると、以下のような問題が発生することがありました。

```shell
    $ gallery-desktop
    gallery-desktop: /usr/lib/x86_64-linux-gnu/libQt5Core.so.5: version
`Qt_5.7' not found (required by gallery-desktop)
    gallery-desktop: /usr/lib/x86_64-linux-gnu/libQt5Core.so.5: version
`Qt_5' not found (required by gallery-desktop)
    ...
    gallery-desktop: /usr/lib/x86_64-linux-gnu/libQt5Core.so.5: version
`Qt_5' not found (required by /usr/lib/libgallery-core.so.1)
```

何が起こったのでしょうか？依存関係を apt-get install -f でインストールしました！ここで、Linux のパッケージ管理の大きな問題点に遭遇しました。.debで指定した依存関係は、特定のバージョンのQtを参照している可能性がありますが、実際にはアップストリームで管理されているパッケージのバージョンに依存しています。つまり、Qtの新しいバージョンがリリースされるたびに、ディストリビューションのメンテナ（UbuntuやFedoraなど）は、公式リポジトリで利用できるようにするためにQtをリパッケージしなければなりません。これは長いプロセスになる可能性があり、メンテナは膨大な数のパッケージを移植しなければなりません!

私たちが述べていることに自信を持つために、lddコマンドを使ってgallery-desktopのライブラリ依存関係を見てみましょう。

```shell
    $ ldd /usr/bin/gallery-desktop
    libgallery-core.so.1 => /usr/lib/libgallery-core.so.1
(0x00007f8110775000)
    libQt5Widgets.so.5 => /usr/lib/x86_64-linux-gnu/libQt5Widgets.so.5
(0x00007f81100e8000)
    libQt5Gui.so.5 => /usr/lib/x86_64-linux-gnu/libQt5Gui.so.5
(0x00007f810fb9f000)
    libQt5Core.so.5 => /usr/lib/x86_64-linux-gnu/libQt5Core.so.5
(0x00007f810f6c9000)
    ...
    libXext.so.6 => /usr/lib/x86_64-linux-gnu/libXext.so.6
(0x00007f810966e000)
```

ご覧のように、libgallery-core.so は /usr/lib で正しく解決され、Qt の依存関係も /usr/lib/x86_64-linux-gnu で解決されています。しかし、Qt はどのバージョンを使っているのでしょうか？その答えはライブラリの詳細にあります。

```shell
    $ ll /usr/lib/x86_64-linux-gnu/libQt5Core.*
    -rw-r--r-- 1 root root 1014 may 2 15:37 libQt5Core.prl
    lrwxrwxrwx 1 root root 19 may 2 15:39 libQt5Core.so ->
libQt5Core.so.5.5.1
    lrwxrwxrwx 1 root root 19 may 2 15:39 libQt5Core.so.5 ->
libQt5Core.so.5.5.1
    lrwxrwxrwx 1 root root 19 may 2 15:39 libQt5Core.so.5.5 ->
libQt5Core.so.5.5.1
    -rw-r--r-- 1 root root 5052920 may 2 15:41 libQt5Core.so.5.5.1
```

libQt5Core.soファイルはlibQt5Core.so.5.5.1へのソフトリンクで、Qtのシステムバージョンは5.5.1であるのに対し、gallery-desktopはQt5.7に依存しています。システムの Qt が Qt インストールを指すようにシステムを設定することができます (Qt インストーラで行います)。しかし、あなたの顧客がギャラリーデスクトップを動かすためだけに Qt を手でインストールする可能性は非常に低いです。

さらに悪いことに、古いバージョンのディストリビューションでは、通常、時間が経ってもパッケージは全く更新されません。Ubuntu 14.04 に Qt 5.7 の Debian パッケージをインストールしてみるだけで、物事がいかに複雑になるかを理解できるでしょう。互換性のない依存関係については触れていません。特定のバージョンの libsqlite3-dev に依存していて、別のアプリケーションが別のものを必要としている場合、物事は醜くなり、1つだけが生き残ることができます。

Linux パッケージは、公式リポジトリで利用できるようにしたい場合や、特定のニーズがある場合には、多くの利点があります。公式リポジトリを使うことは、Linux にアプリケーションをインストールする一般的な方法であり、ユーザが混乱することはありません。Qt のバージョンを Linux ディストリビューションで配布されているものに限定することができれば、それは良い解決策になるかもしれません。

複数のディストリビューションをサポートし、システムを壊さずに依存関係を処理し、アプリケーションが十分に古い依存関係を持っていることを確認しなければなりません。

心配しないでください、すべてが失われているわけではありません。賢い人たちはすでに Linux 上でこの問題を自己完結型のパッケージで解決しています。実際のところ、私たちは自己完結型パッケージを取り上げます。

***

**[戻る](../index.html)**
